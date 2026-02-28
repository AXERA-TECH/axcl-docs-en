# Quick Start

## Model Benchmark

After `AXCL` is installed, the model benchmarking tool `axcl_run_model` is ready to use. This tool has many parameters, which can be viewed with `axcl_run_model --help`. If you're interested in the implementation, you can check the corresponding source code in the `sample` directory. This tool, along with other `cv & llm samples`, is provided in source code form to help users understand the usage of the `API`.

To test the running speed of a model, use `axcl_run_model -m your_model.axmodel -r 100` where `-m` specifies the model to run and `-r` specifies the number of repetitions to benchmark the model speed.

```bash
/root # axcl_run_model -m yolov5s.axmodel -r 100
   Run AxModel:
         model: /opt/data/npu/models/yolov5s.axmodel
          type: 1 Core
          vnpu: Disable
        warmup: 1
        repeat: 100
         batch: { auto: 1 }
    axclrt ver: 1.0.0
   pulsar2 ver: 1.2-patch2 7e6b2b5f
      tool ver: 0.0.1
      cmm size: 12730188 Bytes
  ---------------------------------------------------------------------------
  min =   7.793 ms   max =   7.929 ms   avg =   7.804 ms  median =   7.799 ms
   5% =   7.796 ms   90% =   7.808 ms   95% =   7.832 ms     99% =   7.929 ms
  ---------------------------------------------------------------------------

```

From the running example above, the results not only show the model execution time, but also indicate the toolchain version, model type, and other information.

## CV Examples

### Classification Model

The sample code for the classification model with complete comments is as follows:
```c++
#include "common/args.hpp"            // Header for command line argument parsing
#include "common/types.hpp"           // Header for common type definitions
#include "common/sampler.hpp"         // Header for image preprocessing
#include "common/topk.hpp"            // Header for getting Top K classes
#include "common/imagenet.hpp"        // Header for ImageNet dataset class definitions

#include "utilities/timer.hpp"        // Timer class
#include "utilities/file.hpp"         // File operation utilities
#include "utilities/scalar_guard.hpp" // Resource management class for ensuring resource release
#include "utilities/vector_guard.hpp" // Resource management class for dynamic arrays

#include <axcl.h>                     // AXCL API header
#include <cmdline.h>                  // Command line parsing library
#include <opencv2/opencv.hpp>         // OpenCV image processing library

// Set default config file path
constexpr char  DEFAULT_CONFIG[] = "/usr/local/axcl/axcl.json";

// Set default input image height and width
constexpr int   DEFAULT_IMG_H = 224;
constexpr int   DEFAULT_IMG_W = 224;

// Set default loop count
constexpr int   DEFAULT_LOOP_COUNT = 1;

// Set default Top K value
constexpr int   DEFAULT_TOP_K = 5;

int main(int argc, char* argv[]) {
    cmdline::parser cmd; // Create command line parser

    // Add command line arguments
    cmd.add<std::string>("config", 'c', "config file: axcl.json", false, DEFAULT_CONFIG); // Config file path
    cmd.add<int>("device", 'd', "device index (if multi-card available)", false, 0, cmdline::range(0, 3)); // Device index

#if defined(ENV_CHIP_SERIES_MC50)
    cmd.add<int>("kind", 'k', "visual npu kind", false, 0, cmdline::range(0, 3)); // NPU type
#endif

#if defined(ENV_CHIP_SERIES_MC20E)
    cmd.add<int>("kind", 'k', "visual npu enable flag", false, 0, cmdline::range(0, 1)); // NPU enable flag
#endif

    cmd.add<std::string>("model", 'm', "model file(a.k.a. .axmodel)", true, ""); // Required: model file
    cmd.add<std::string>("image", 'i', "image file", true, ""); // Required: image file
    cmd.add<std::string>("size", 'g', "input_h,input_w", false, std::to_string(DEFAULT_IMG_H) + "," + std::to_string(DEFAULT_IMG_W)); // Input image size

    cmd.add("swap-rb", 's', "swap rgb(bgr) to bgr(rgb)"); // Optional: swap color channels

    cmd.add<int>("topk", 't', "top k values", false, DEFAULT_TOP_K, cmdline::range(1, static_cast<int>(std::size(IMAGENET_CLASSES)))); // Top K value
    cmd.add<int>("repeat", 'r', "repeat count", false, DEFAULT_LOOP_COUNT, cmdline::range(1, std::numeric_limits<int>::max())); // Repeat count
    cmd.parse_check(argc, argv); // Parse and check command line arguments

    // 0-0. Check config file
    if (const auto config = cmd.get<std::string>("config");
        cmd.exist("config") && (!utilities::exists(config) || !utilities::is_regular_file(config))) {
        fprintf(stderr, "Config file %s is not exist, please check it.\n", config.c_str()); // Print error message
        return -1; // Exit program
    }

    // 0-1. Check if model and image files exist
    const auto model_file = cmd.get<std::string>("model");
    const auto image_file = cmd.get<std::string>("image");

    if (const auto model_file_flag = utilities::exists(model_file), image_file_flag = utilities::exists(image_file);
        !model_file_flag | !image_file_flag) {
        auto show_error = [](const std::string& kind, const std::string& value) {
            fprintf(stderr, "Input file %s(%s) is not exist, please check it.\n", kind.c_str(), value.c_str());
        };

        if (!model_file_flag) { show_error("model", model_file); } // Model file not found error
        if (!image_file_flag) { show_error("image", image_file); } // Image file not found error

        return -1; // Exit program
    }

    // 0-2. Get input image size
    std::array<int, 2> input_size = {DEFAULT_IMG_H, DEFAULT_IMG_W}; {
        auto input_size_string = cmd.get<std::string>("size");
        if (auto input_size_flag = common::parse_args(input_size_string, input_size); !input_size_flag) {
            auto show_error = [](const std::string& kind, const std::string& value) {
                fprintf(stderr, "Input %s(%s) is not allowed, please check it.\n", kind.c_str(), value.c_str());
            };

            show_error("size", input_size_string); // Input size error
            return -1; // Exit program
        }
    }

    // 0-3. Get repeat count
    auto repeat = cmd.get<int>("repeat");

    // 1-0. Get device index
    const auto device_index = static_cast<uint32_t>(cmd.get<int>("device"));

    // 1-1 Initialize AXCL, using scalar_guard to ensure final resource release
    fprintf(stdout, "axcl initializing...\n");
    auto env_guard = utilities::scalar_guard<int32_t>(
        axclInit(cmd.exist("config") ? cmd.get<std::string>("config").c_str() : nullptr), // Initialize AXCL
        [](const int32_t& code) {
            if (0 == code) {
                std::ignore = axclFinalize(); // Ensure cleanup is called if initialization fails
            }
        }
    );

    // 1-2. Check initialization result
    if (const int ret = env_guard.get(); 0 != ret) {
        fprintf(stderr, "Init axcl failed{0x%08X}.\n", ret); // Print initialization failure info
        return false; // Exit program
    }
    fprintf(stdout, "axcl inited.\n");

    // 1-3. Get device list
    axclrtDeviceList lst;
    if (const auto ret = axclrtGetDeviceList(&lst); 0 != ret || 0 == lst.num) {
        fprintf(stderr,
            "Get axcl device failed{0x%08X}, find total %d device.\n", ret, lst.num); // Print device retrieval failure
        return false; // Exit program
    }

    // 1-4. Check device index
    if (device_index >= lst.num) {
        fprintf(stderr,
            "Specified device index{%u} is out of range{total %d}.\n", device_index, lst.num); // Print index out of range
        return false; // Exit program
    }

    // 1-5. Set device
    if (const auto ret = axclrtSetDevice(lst.devices[device_index]); 0 != ret) {
        fprintf(stderr, "Set axcl device as index{%u} failed{0x%08X}.\n", device_index, ret); // Print set device failure
        return false; // Exit program
    }
    fprintf(stdout,"Select axcl device{index: %u} as {%d}.\n", device_index, lst.devices[device_index]);

    // 1-6. Initialize NPU
    const int kind = cmd.get<int>("kind"); // Get NPU type
    if (const int ret = axclrtEngineInit(static_cast<axclrtEngineVNpuKind>(kind)); 0 != ret) {
        fprintf(stderr, "Init axclrt Engine as kind{%s} failed{0x%08X}.\n", common::get_visual_mode_string(kind).c_str(), ret); // Print NPU init failure
        return false; // Exit program
    }
    fprintf(stdout, "axclrt Engine inited.\n");

    // 1-7. Print parameter info
    fprintf(stdout, "--------------------------------------\n");
    fprintf(stdout, "model file : %s\n", model_file.c_str());
    fprintf(stdout, "image file : %s\n", image_file.c_str());
    fprintf(stdout, "img height : %d\n", input_size[0]);
    fprintf(stdout, "img width  : %d\n", input_size[1]);
    fprintf(stdout, "--------------------------------------\n");

    // 2-1. Load model
    auto m = utilities::scalar_guard<uint64_t>(
        [&model_file]() {
            if (uint64_t id; 0 == axclrtEngineLoadFromFile(model_file.c_str(), &id)) { // Load model from file
                return id; // Return model ID
            }
            fprintf(stderr, "Create model{%s} handle failed.\n", model_file.c_str());
            return uint64_t{0}; // Return 0 on failure
        },
        [](const uint64_t& id) {
            if (uint64_t{0} != id) {
                std::ignore = axclrtEngineUnload(id); // Unload model
            }
        }
    );

    // 2-2. Create context
    uint64_t ctx = 0;
    if (const auto ret = axclrtEngineCreateContext(m.get(), &ctx); 0 != ret) {
        fprintf(stderr, "Create model{%s} context failed.\n", model_file.c_str()); // Print context creation failure
        return false; // Exit program
    }

    // 2-3. Get IO info
    auto info = utilities::scalar_guard<axclrtEngineIOInfo>(
        [&m]() {
            axclrtEngineIOInfo i;
            if (0 == axclrtEngineGetIOInfo(m.get(), &i)) {
                return i; // Return IO info
            }
            fprintf(stderr, "Get model io info failed.\n");
            return axclrtEngineIOInfo{}; // Return empty struct on failure
        },
        [](const axclrtEngineIOInfo& i) {
            std::ignore = axclrtEngineDestroyIOInfo(i); // Release IO info
        }
    );

    // 2-4. Create IO
    auto io = utilities::scalar_guard<axclrtEngineIO>(
        [&info]() {
            axclrtEngineIO i;
            if (0 == axclrtEngineCreateIO(info.get(), &i)) {
                return i; // Return created IO
            }
            fprintf(stderr, "Create model io failed.\n");
            return axclrtEngineIO{}; // Return empty struct on failure
        },
        [](const axclrtEngineIO& i) {
            std::ignore = axclrtEngineDestroyIO(i); // Release IO
        }
    );

    // 2-5. Get input count
    uint32_t input_count = 0;
    if (input_count = axclrtEngineGetNumInputs(info.get()); 0 == input_count) {
        fprintf(stderr, "Get model input count failed.\n"); // Print get input count failure
        return false; // Exit program
    }

    // 2-6. Get input sizes
    std::vector<uint32_t> inputs_size(input_count, 0);
    for (uint32_t i = 0; i < input_count; i++) {
        inputs_size[i] = axclrtEngineGetInputSizeByIndex(info.get(), 0, i); // Get size of each input
    }

    // 2-7. Allocate input memory
    auto inputs = utilities::vector_guard<void*>(
        [&input_count, &inputs_size]() {
            std::vector<void*> ptrs(input_count, nullptr);
            for (uint32_t i = 0; i < input_count; i++) {
                if (const auto ret = axclrtMalloc(&ptrs[i], inputs_size[i], axclrtMemMallocPolicy{}); 0 != ret) {
                    fprintf(stderr, "Memory allocation for input tensor{index: %d} failed{0x%08X}.\n", i, ret); // Print memory allocation failure
                }
            }
            return ptrs; // Return allocated memory pointer array
        },
        [](void* ptr) {
            if (nullptr != ptr) {
                std::ignore = axclrtFree(ptr); // Free memory
                ptr = nullptr;
            }
        }
    );

    // 2-8. Check if input memory allocation succeeded
    for (uint32_t i = 0; i < input_count; i++) {
        if (nullptr == inputs.get()[i]) {
            return false; // Exit program
        }
    }

    // 2-9. Set input buffers
    for (uint32_t i = 0; i < input_count; i++) {
        if (const auto ret = axclrtEngineSetInputBufferByIndex(io.get(), i, inputs.get()[i], inputs_size[i]); 0 != ret) {
            fprintf(stderr, "Set input buffer{index: %d} failed{0x%08X}.\n", i, ret); // Print set input buffer failure
            return false; // Exit program
        }
    }

    // 2-10. Get output count
    uint32_t output_count = 0;
    if (output_count = axclrtEngineGetNumOutputs(info.get()); 0 == output_count) {
        fprintf(stderr, "Get model output count failed.\n"); // Print get output count failure
        return false; // Exit program
    }

    // 2-11. Get output sizes
    std::vector<uint32_t> outputs_size(output_count, 0);
    for (uint32_t i = 0; i < output_count; i++) {
        outputs_size[i] = axclrtEngineGetOutputSizeByIndex(info.get(), 0, i); // Get size of each output
    }

    // 2-12. Allocate output memory
    auto outputs = utilities::vector_guard<void*>(
        [&output_count, &outputs_size]() {
            std::vector<void*> ptrs(output_count, nullptr);
            for (uint32_t i = 0; i < output_count; i++) {
                if (const auto ret = axclrtMalloc(&ptrs[i], outputs_size[i], axclrtMemMallocPolicy{}); 0 != ret) {
                    fprintf(stderr, "Memory allocation for output tensor{index: %d} failed{0x%08X}.\n", i, ret); // Print memory allocation failure
                }
            }
            return ptrs; // Return allocated memory pointer array
        },
        [](void* ptr) {
            if (nullptr != ptr) {
                std::ignore = axclrtFree(ptr); // Free memory
                ptr = nullptr;
            }
        }
    );

    // 2-13. Check if output memory allocation succeeded
    for (uint32_t i = 0; i < output_count; i++) {
        if (nullptr == outputs.get()[i]) {
            return false; // Exit program
        }
    }

    // 2-14. Set output buffers
    for (uint32_t i = 0; i < output_count; i++) {
        if (const auto ret = axclrtEngineSetOutputBufferByIndex(io.get(), i, outputs.get()[i], outputs_size[i]); 0 != ret) {
            fprintf(stderr, "Set output buffer{index: %d} failed{0x%08X}.\n", i, ret); // Print set output buffer failure
            return false; // Exit program
        }
    }

    // 3-0. Read input image
    cv::Mat src = cv::imread(image_file);
    if (src.empty()) {
        fprintf(stderr, "Read image failed.\n"); // Print read image failure
        return -1; // Exit program
    }

    // 3-1. Allocate input host buffer
    std::vector<uint8_t> input_buffer(input_size[1] * input_size[0] * 3, 0); // Allocate input buffer (RGB)
    auto dst = cv::Mat(cv::Size(input_size[1], input_size[0]), CV_8UC3, input_buffer.data()); // Create OpenCV Mat

    // 3-2. Preprocess image (resize and swap RGB channels)
    preprocess::sampler sampler(preprocess::crop_type::imagenet, preprocess::resize_type::linear, cmd.exist("swap-rb"));
    sampler(src, dst); // Process input image

    // 3-3. Send input data to device
    if (const auto ret = axclrtMemcpy(inputs.get()[0], input_buffer.data(), input_buffer.size(), AXCL_MEMCPY_HOST_TO_DEVICE); 0 != ret) {
        fprintf(stderr, "Copy input data to device failed{0x%08X}.\n", ret); // Print data copy failure
        return false; // Exit program
    }

    // 3-4. Execute model inference
    std::vector<float> time_costs(repeat, 0); // Record inference time for each run
    for (int i = 0; i < repeat; ++i) {
        utilities::timer tick; // Create timer
        const auto ret = axclrtEngineExecute(m.get(), ctx, 0, io.get()); // Execute inference
        time_costs[i] = tick.elapsed(); // Record inference time
        if ( 0 != ret) {
            fprintf(stderr, "Run model failed{0x%08X}.\n", ret); // Print inference failure
            return false; // Exit program
        }
    }

    // 3-5. Get output results
    std::vector<float> output_buffer(outputs_size[0], 0.f); // Initialize output buffer
    if (const auto ret = axclrtMemcpy(output_buffer.data(), outputs.get()[0], outputs_size[0], AXCL_MEMCPY_DEVICE_TO_HOST); 0 != ret) {
        fprintf(stderr, "Copy output data to host failed{0x%08X}.\n", ret); // Print data copy failure
        return false; // Exit program
    }

    // 3-6. Post-process, extract results
    const auto topK = classification::topK(output_buffer.data(), output_buffer.size(), cmd.get<int>("topk")); // Get Top K results
    for (const auto& [prob, index] : topK) {
        fprintf(stdout, "%3d: %4.1f%%,  %s\n", index, prob, IMAGENET_CLASSES[index]); // Print probability and class name
    }
    fprintf(stdout, "--------------------------------------\n");

    return 0; // Normal program exit
}
```

Using `imagenet_cat.jpg` from the [`ImageNet`](https://image-net.org/) dataset as the classification object, the sample produces the following output (note: the model and input image should be adjusted based on your actual setup):

![](../res/imagenet_cat.jpg)

```bash
/root # /opt/bin/axcl/axcl_sample_classification -m /opt/data/npu/models/mobilenetv2.axmodel -i /opt/data/npu/images/cat.jpg
axcl initializing...
axcl inited.
Select axcl device{index: 0} as {129}.
axclrt Engine inited.
--------------------------------------
model file : /opt/data/npu/models/mobilenetv2.axmodel
image file : /opt/data/npu/images/cat.jpg
img height : 224
img width  : 224
--------------------------------------
282:  9.8%,  tiger cat
285:  9.8%,  Egyptian cat
283:  9.5%,  Persian cat
281:  9.4%,  tabby, tabby cat
463:  7.5%,  bucket, pail
--------------------------------------
```

As we can see, the cat is classified as tiger cat for Top1; and the Top5 classification is basically correct.

### Detection Model

The sample code for the detection model with complete comments is as follows:
```c++
#include "common/args.hpp"            // Header for command line argument parsing
#include "common/types.hpp"           // Header for common type definitions
#include "common/sampler.hpp"         // Header for image preprocessing
#include "common/detection.hpp"       // Header for object detection data structures and algorithms
#include "common/mscoco.hpp"          // Header for MS COCO dataset class definitions
#include "common/palette.hpp"         // Header for palette generation and color conversion

#include "utilities/timer.hpp"        // Timer class for time measurement
#include "utilities/file.hpp"         // File operation utilities
#include "utilities/scalar_guard.hpp" // Resource management class for ensuring resource release
#include "utilities/vector_guard.hpp" // Resource management class for dynamic arrays

#include <axcl.h>                     // AXCL API header

#include <cmdline.h>                  // Command line parsing library
#include <opencv2/opencv.hpp>         // OpenCV image processing library

#include <algorithm>                  // Algorithm functions
#include <numeric>                    // Numeric computation functions

// Set default config file path
constexpr char  DEFAULT_CONFIG[] = "/usr/local/axcl/axcl.json";

// Set default input image height and width
constexpr int   DEFAULT_IMG_H = 640;
constexpr int   DEFAULT_IMG_W = 640;
// Set default probability threshold and IOU threshold
constexpr float DEFAULT_PROB_THRESHOLD = 0.45f;
constexpr float DEFAULT_IOU_THRESHOLD = 0.45f;

// Set default loop count
constexpr int   DEFAULT_LOOP_COUNT = 1;

// Define anchor boxes
const std::vector<std::vector<detection::box>> anchors{
    {{10, 13}, {16, 30}, {33, 23}},
    {{30, 61}, {62, 45}, {59, 119}},
    {{116, 90}, {156, 198}, {373, 326}}
};

// Define stride for model output head downsampling
const std::vector<int> strides{8, 16, 32};
// Generate color palette
std::vector<cv::Vec3b> palette = common::palette(std::size(MSCOCO_CLASSES));

// Draw detected objects
static void draw_objects(const cv::Mat& bgr, const std::string& name, const std::vector<detection::object>& objects) {
    // Clone input image for drawing
    cv::Mat image = bgr.clone();

    // Iterate over each detected object
    for (const auto& [rect, prob, label] : objects) {
        const auto& [x, y, w, h] = rect; // Extract rectangle coordinates and size
        fprintf(stdout, "%2d: %3.0f%%, [%4.0f, %4.0f, %4.0f, %4.0f], %s\n",
            label, prob * 100, x, y, x + w, y + h, MSCOCO_CLASSES[label]); // Print object info

        const cv::Rect obj_rect{static_cast<int>(x), static_cast<int>(y), static_cast<int>(w), static_cast<int>(h)}; // Create rectangle object

        const auto& color = palette[label]; // Get color for this object
        cv::rectangle(image, obj_rect, color, 2); // Draw rectangle on image

        char text[256];
        sprintf(text, "%s %.1f%%", MSCOCO_CLASSES[label], prob * 100); // Prepare display text

        // Calculate text position
        int baseLine = 0;
        const cv::Size label_size = cv::getTextSize(text, cv::FONT_HERSHEY_TRIPLEX, 0.5, 1, &baseLine);

        int cv_x = std::max(static_cast<int>(x - 1), 0); // Ensure text doesn't go beyond image boundary
        int cv_y = std::max(static_cast<int>(y + 1) - label_size.height - baseLine, 0);

        if (cv_x + label_size.width > image.cols) {
            cv_x = image.cols - label_size.width; // Adjust text x coordinate
        }
        if (cv_y + label_size.height > image.rows) {
            cv_y = image.rows - label_size.height; // Adjust text y coordinate
        }

        // Draw text background
        const cv::Size text_background_size(label_size.width, label_size.height + baseLine);
        const cv::Rect text_background(cv::Point(cv_x, cv_y), text_background_size);
        cv::rectangle(image, text_background, color, -1); // Draw filled rectangle

        const cv::Point text_origin(cv_x, cv_y + label_size.height); // Text starting position
        const auto text_color = common::get_complementary_color(color); // Get complementary color
        cv::putText(image, text, text_origin, cv::FONT_HERSHEY_TRIPLEX, 0.5, text_color); // Draw text on image
    }

    // Save result image to file
    cv::imwrite(std::string(name) + ".jpg", image);
}

int main(int argc, char* argv[]) {
    cmdline::parser cmd; // Create command line parser

    // Add command line arguments
    cmd.add<std::string>("config", 'c', "config file: axcl.json", false, DEFAULT_CONFIG);
    cmd.add<int>("device", 'd', "device index (if multi-card available)", false, 0, cmdline::range(0, 3));

#if defined(ENV_CHIP_SERIES_MC50)
    cmd.add<int>("kind", 'k', "visual npu kind", false, 0, cmdline::range(0, 3));
#endif

#if defined(ENV_CHIP_SERIES_MC20E)
    cmd.add<int>("kind", 'k', "visual npu enable flag", false, 0, cmdline::range(0, 1));
#endif

    cmd.add<std::string>("model", 'm', "model file(a.k.a. .axmodel)", true, ""); // Required: model file
    cmd.add<std::string>("image", 'i', "image file", true, ""); // Required: image file
    cmd.add<std::string>("size", 'g', "input_h,input_w", false, std::to_string(DEFAULT_IMG_H) + "," + std::to_string(DEFAULT_IMG_W)); // Input image size

    cmd.add("swap-rb", 's', "swap rgb(bgr) to bgr(rgb)"); // Optional: swap color channels

    cmd.add<int>("repeat", 'r', "repeat count", false, DEFAULT_LOOP_COUNT, cmdline::range(1, std::numeric_limits<int>::max())); // Repeat count
    cmd.parse_check(argc, argv); // Parse and check command line arguments

    // 0-0. Check config file
    if (const auto config = cmd.get<std::string>("config");
        cmd.exist("config") && (!utilities::exists(config) || !utilities::is_regular_file(config))) {
        fprintf(stderr, "Config file %s is not exist, please check it.\n", config.c_str()); // Print error message
        return -1; // Exit program
    }

    // 0-1. Check if model and image files exist
    const auto model_file = cmd.get<std::string>("model");
    const auto image_file = cmd.get<std::string>("image");

    if (const auto model_file_flag = utilities::exists(model_file), image_file_flag = utilities::exists(image_file);
        !model_file_flag | !image_file_flag) {
        auto show_error = [](const std::string& kind, const std::string& value) {
            fprintf(stderr, "Input file %s(%s) is not exist, please check it.\n", kind.c_str(), value.c_str());
        };

        if (!model_file_flag) { show_error("model", model_file); } // Model file not found error
        if (!image_file_flag) { show_error("image", image_file); } // Image file not found error

        return -1; // Exit program
    }

    // 0-2. Get input image size
    std::array<int, 2> input_size = {DEFAULT_IMG_H, DEFAULT_IMG_W}; {
        auto input_size_string = cmd.get<std::string>("size");
        if (auto input_size_flag = common::parse_args(input_size_string, input_size); !input_size_flag) {
            auto show_error = [](const std::string& kind, const std::string& value) {
                fprintf(stderr, "Input %s(%s) is not allowed, please check it.\n", kind.c_str(), value.c_str());
            };

            show_error("size", input_size_string); // Input size error
            return -1; // Exit program
        }
    }

    // 0-3. Get repeat count
    auto repeat = cmd.get<int>("repeat");

    // 1-0. Get device index
    const auto device_index = static_cast<uint32_t>(cmd.get<int>("device"));

    // 1-1 Initialize AXCL, using scalar_guard to ensure final resource release
    fprintf(stdout, "axcl initializing...\n");
    auto env_guard = utilities::scalar_guard<int32_t>(
        axclInit(cmd.exist("config") ? cmd.get<std::string>("config").c_str() : nullptr), // Initialize AXCL
        [](const int32_t& code) {
            if (0 == code) {
                std::ignore = axclFinalize(); // Ensure cleanup is called if initialization fails
            }
        }
    );

    // 1-2. Check initialization result
    if (const int ret = env_guard.get(); 0 != ret) {
        fprintf(stderr, "Init axcl failed{0x%08X}.\n", ret); // Print initialization failure
        return false; // Exit program
    }
    fprintf(stdout, "axcl inited.\n");

    // 1-3. Get device list
    axclrtDeviceList lst;
    if (const auto ret = axclrtGetDeviceList(&lst); 0 != ret || 0 == lst.num) {
        fprintf(stderr,
            "Get axcl device failed{0x%08X}, find total %d device.\n", ret, lst.num); // Print device retrieval failure
        return false; // Exit program
    }

    // 1-4. Check device index
    if (device_index >= lst.num) {
        fprintf(stderr,
            "Specified device index{%u} is out of range{total %d}.\n", device_index, lst.num); // Print index out of range
        return false; // Exit program
    }

    // 1-5. Set device
    if (const auto ret = axclrtSetDevice(lst.devices[device_index]); 0 != ret) {
        fprintf(stderr, "Set axcl device as index{%u} failed{0x%08X}.\n", device_index, ret); // Print set device failure
        return false; // Exit program
    }
    fprintf(stdout,"Select axcl device{index: %u} as {%d}.\n", device_index, lst.devices[device_index]);

    // 1-6. Initialize NPU
    const int kind = cmd.get<int>("kind"); // Get NPU type
    if (const int ret = axclrtEngineInit(static_cast<axclrtEngineVNpuKind>(kind)); 0 != ret) {
        fprintf(stderr, "Init axclrt Engine as kind{%s} failed{0x%08X}.\n", common::get_visual_mode_string(kind).c_str(), ret); // Print NPU init failure
        return false; // Exit program
    }
    fprintf(stdout, "axclrt Engine inited.\n");

    // 1-7. Print parameter info
    fprintf(stdout, "--------------------------------------\n");
    fprintf(stdout, "model file : %s\n", model_file.c_str());
    fprintf(stdout, "image file : %s\n", image_file.c_str());
    fprintf(stdout, "img height : %d\n", input_size[0]);
    fprintf(stdout, "img width  : %d\n", input_size[1]);
    fprintf(stdout, "--------------------------------------\n");

    // 2-1. Load model
    auto m = utilities::scalar_guard<uint64_t>(
        [&model_file]() {
            if (uint64_t id; 0 == axclrtEngineLoadFromFile(model_file.c_str(), &id)) { // Load model from file
                return id; // Return model ID
            }
            fprintf(stderr, "Create model{%s} handle failed.\n", model_file.c_str());
            return uint64_t{0}; // Return 0 on failure
        },
        [](const uint64_t& id) {
            if (uint64_t{0} != id) {
                std::ignore = axclrtEngineUnload(id); // Unload model
            }
        }
    );

    // 2-2. Create context
    uint64_t ctx = 0;
    if (const auto ret = axclrtEngineCreateContext(m.get(), &ctx); 0 != ret) {
        fprintf(stderr, "Create model{%s} context failed.\n", model_file.c_str()); // Print context creation failure
        return false; // Exit program
    }

    // 2-3. Get IO info
    auto info = utilities::scalar_guard<axclrtEngineIOInfo>(
        [&m]() {
            axclrtEngineIOInfo i;
            if (0 == axclrtEngineGetIOInfo(m.get(), &i)) {
                return i; // Return IO info
            }
            fprintf(stderr, "Get model io info failed.\n");
            return axclrtEngineIOInfo{}; // Return empty struct on failure
        },
        [](const axclrtEngineIOInfo& i) {
            std::ignore = axclrtEngineDestroyIOInfo(i); // Release IO info
        }
    );

    // 2-4. Create IO
    auto io = utilities::scalar_guard<axclrtEngineIO>(
        [&info]() {
            axclrtEngineIO i;
            if (0 == axclrtEngineCreateIO(info.get(), &i)) {
                return i; // Return created IO
            }
            fprintf(stderr, "Create model io failed.\n");
            return axclrtEngineIO{}; // Return empty struct on failure
        },
        [](const axclrtEngineIO& i) {
            std::ignore = axclrtEngineDestroyIO(i); // Release IO
        }
    );

    // 2-5. Get input count
    uint32_t input_count = 0;
    if (input_count = axclrtEngineGetNumInputs(info.get()); 0 == input_count) {
        fprintf(stderr, "Get model input count failed.\n"); // Print get input count failure
        return false; // Exit program
    }

    // 2-6. Get input sizes
    std::vector<uint32_t> inputs_size(input_count, 0);
    for (uint32_t i = 0; i < input_count; i++) {
        inputs_size[i] = axclrtEngineGetInputSizeByIndex(info.get(), 0, i); // Get size of each input
    }

    // 2-7. Allocate input memory
    auto inputs = utilities::vector_guard<void*>(
        [&input_count, &inputs_size]() {
            std::vector<void*> ptrs(input_count, nullptr);
            for (uint32_t i = 0; i < input_count; i++) {
                if (const auto ret = axclrtMalloc(&ptrs[i], inputs_size[i], axclrtMemMallocPolicy{}); 0 != ret) {
                    fprintf(stderr, "Memory allocation for input tensor{index: %d} failed{0x%08X}.\n", i, ret); // Print memory allocation failure
                }
            }
            return ptrs; // Return allocated memory pointer array
        },
        [](void* ptr) {
            if (nullptr != ptr) {
                std::ignore = axclrtFree(ptr); // Free memory
                ptr = nullptr;
            }
        }
    );

    // 2-8. Check if input memory allocation succeeded
    for (uint32_t i = 0; i < input_count; i++) {
        if (nullptr == inputs.get()[i]) {
            return false; // Exit program
        }
    }

    // 2-9. Set input buffers
    for (uint32_t i = 0; i < input_count; i++) {
        if (const auto ret = axclrtEngineSetInputBufferByIndex(io.get(), i, inputs.get()[i], inputs_size[i]); 0 != ret) {
            fprintf(stderr, "Set input buffer{index: %d} failed{0x%08X}.\n", i, ret); // Print set input buffer failure
            return false; // Exit program
        }
    }

    // 2-10. Get output count
    uint32_t output_count = 0;
    if (output_count = axclrtEngineGetNumOutputs(info.get()); 0 == output_count) {
        fprintf(stderr, "Get model output count failed.\n"); // Print get output count failure
        return false; // Exit program
    }

    // 2-11. Get output sizes
    std::vector<uint32_t> outputs_size(output_count, 0);
    for (uint32_t i = 0; i < output_count; i++) {
        outputs_size[i] = axclrtEngineGetOutputSizeByIndex(info.get(), 0, i); // Get size of each output
    }

    // 2-12. Allocate output memory
    auto outputs = utilities::vector_guard<void*>(
        [&output_count, &outputs_size]() {
            std::vector<void*> ptrs(output_count, nullptr);
            for (uint32_t i = 0; i < output_count; i++) {
                if (const auto ret = axclrtMalloc(&ptrs[i], outputs_size[i], axclrtMemMallocPolicy{}); 0 != ret) {
                    fprintf(stderr, "Memory allocation for output tensor{index: %d} failed{0x%08X}.\n", i, ret); // Print memory allocation failure
                }
            }
            return ptrs; // Return allocated memory pointer array
        },
        [](void* ptr) {
            if (nullptr != ptr) {
                std::ignore = axclrtFree(ptr); // Free memory
                ptr = nullptr;
            }
        }
    );

    // 2-13. Check if output memory allocation succeeded
    for (uint32_t i = 0; i < output_count; i++) {
        if (nullptr == outputs.get()[i]) {
            return false; // Exit program
        }
    }

    // 2-14. Set output buffers
    for (uint32_t i = 0; i < output_count; i++) {
        if (const auto ret = axclrtEngineSetOutputBufferByIndex(io.get(), i, outputs.get()[i], outputs_size[i]); 0 != ret) {
            fprintf(stderr, "Set output buffer{index: %d} failed{0x%08X}.\n", i, ret); // Print set output buffer failure
            return false; // Exit program
        }
    }

    // 3-0. Read input image
    cv::Mat src = cv::imread(image_file);
    if (src.empty()) {
        fprintf(stderr, "Read image failed.\n"); // Print read image failure
        return -1; // Exit program
    }

    // 3-1. Allocate input host buffer
    std::vector<uint8_t> input_buffer(input_size[1] * input_size[0] * 3, 0); // Allocate input buffer (RGB)
    auto dst = cv::Mat(cv::Size(input_size[1], input_size[0]), CV_8UC3, input_buffer.data()); // Create OpenCV Mat

    // 3-2. Preprocess image (resize and swap RGB channels)
    preprocess::sampler sampler(preprocess::crop_type::letterbox, preprocess::resize_type::linear, cmd.exist("swap-rb"));
    sampler(src, dst); // Process input image

    // 3-3. Send input data to device
    if (const auto ret = axclrtMemcpy(inputs.get()[0], input_buffer.data(), input_buffer.size(), AXCL_MEMCPY_HOST_TO_DEVICE); 0 != ret) {
        fprintf(stderr, "Copy input data to device failed{0x%08X}.\n", ret); // Print data copy failure
        return false; // Exit program
    }

    // 3-4. Execute model inference
    std::vector<float> run_costs(repeat, 0); // Record inference time for each run
    for (int i = 0; i < repeat; ++i) {
        utilities::timer tick; // Create timer
        const auto ret = axclrtEngineExecute(m.get(), ctx, 0, io.get()); // Execute inference
        run_costs[i] = tick.elapsed(); // Record inference time
        if ( 0 != ret) {
            fprintf(stderr, "Run model failed{0x%08X}.\n", ret); // Print inference failure
            return false; // Exit program
        }
    }

    // 3-5. Get output results
    std::vector<float> output_buffer(outputs_size[0], 0.f); // Initialize output buffer
    if (const auto ret = axclrtMemcpy(output_buffer.data(), outputs.get()[0], outputs_size[0], AXCL_MEMCPY_DEVICE_TO_HOST); 0 != ret) {
        fprintf(stderr, "Copy output data to host failed{0x%08X}.\n", ret); // Print data copy failure
        return false; // Exit program
    }

    // 3-6. Post-process, extract detection results
    const auto topK = classification::topK(output_buffer.data(), output_buffer.size(), cmd.get<int>("topk")); // Get Top K results
    for (const auto& [prob, index] : topK) {
        fprintf(stdout, "%3d: %4.1f%%,  %s\n", index, prob, IMAGENET_CLASSES[index]); // Print probability and class name
    }
    fprintf(stdout, "--------------------------------------\n");

    return 0; // Normal program exit
}
```

Using voc_dog.jpg from the [`PASCAL VOC`](http://host.robots.ox.ac.uk/pascal/VOC/) dataset as the detection object, the sample produces the following output (note: the model and input image should be adjusted based on your actual setup):

![](../res/voc_dog.jpg)

```bash
/root # /opt/bin/axcl/axcl_sample_yolov5s -m yolov5s.axmodel -i voc_dog.jpg
axcl initializing...
axcl inited.
Select axcl device{index: 0} as {129}.
axclrt Engine inited.
--------------------------------------
model file : /opt/data/npu/models/yolov5s.axmodel
image file : /opt/data/npu/images/dog.jpg
img height : 640
img width  : 640
--------------------------------------
post process cost time:1.86 ms
--------------------------------------
Repeat 1 times, avg time 8.02 ms, max_time 8.02 ms, min_time 8.02 ms
--------------------------------------
16:  91%, [ 138,  218,  310,  541], dog
 2:  69%, [ 470,   76,  690,  173], car
 1:  56%, [ 158,  120,  569,  420], bicycle
```

As we can see, 3 objects were detected with their class `ID`, confidence, and coordinates. A detection result file named `yolov5s_out.jpg` is saved in the execution directory of the `sample`, which can be previewed with an image viewer.

![](../res/voc_dog_yolov5s_out.jpg)

Using voc_horse.jpg from the [`PASCAL VOC`](http://host.robots.ox.ac.uk/pascal/VOC/) dataset as the detection object, the sample produces the following output (note: the model and input image should be adjusted based on your actual setup):

![](../res/voc_horse.jpg)

```bash
/root # /opt/bin/axcl/axcl_sample_yolov5s -m yolov5s.axmodel -i voc_horse.jpg
axcl initializing...
axcl inited.
Select axcl device{index: 0} as {129}.
axclrt Engine inited.
--------------------------------------
model file : /opt/data/npu/models/yolov5s.axmodel
image file : voc_horse.jpg
img height : 640
img width  : 640
--------------------------------------
post process cost time:1.80 ms
--------------------------------------
Repeat 1 times, avg time 8.02 ms, max_time 8.02 ms, min_time 8.02 ms
--------------------------------------
17:  84%, [ 208,   52,  431,  374], horse
16:  84%, [ 142,  201,  197,  350], dog
 0:  83%, [ 273,   16,  350,  226], person
 7:  74%, [   0,  107,  132,  195], truck
 0:  73%, [ 430,  123,  449,  179], person
 0:  47%, [ 402,  130,  412,  148], person
```

As we can see, 6 objects were detected with their class `ID`, confidence, and coordinates. A detection result file named `yolov5s_out.jpg` is saved in the execution directory of the `sample`, which can be previewed with an image viewer.

![](../res/voc_horse_yolov5s_out.jpg)

# AXCL User Manual

[Web Preview](https://axcl-docs-en.readthedocs.io/en/latest/)

## 1. Project Background

**AXCL** is the Host API for AX650N-based PCIE EP products.

- Provide related APIs and examples for NPU computing with AXCL.
- Facilitate community developers to co-maintain documentation.

## 2. Local Build Guide

### 2.1 git clone

```bash
git clone https://github.com/AXERA-TECH/axcl-docs-en.git
```

The directory tree is as follows:

```bash
.
├── LICENSE
├── Makefile
├── README.md
├── build
│   ├── doctrees
│   └── html
├── requirements.txt
├── res
│   ├── AX-M1.png
│   ├── LLM8850.png
│   ├── ax650_win64_drv_installed.png
│   ├── ax650_win64_drv_uninstall.png
│   ├── ax650_win64_install.mp4
│   ├── ax650_win64_smi.png
│   ├── axcl_architecture.svg
│   ├── axcl_concept.svg
│   ├── centos_dmsg_grep_cma.png
│   ├── centos_grub_info.png
│   ├── centos_selinux.png
│   ├── depth_anything_out.png
│   ├── faq_deb_install_1.png
│   ├── faq_deb_install_2.png
│   ├── faq_deb_install_3.png
│   ├── ftrivial_auto_var_init.png
│   ├── hw-board-id.png
│   ├── imagenet_cat.jpg
│   ├── m.2_rstn.png
│   ├── pci-bus-tupo.png
│   ├── pulsar2.png
│   ├── sample_vdec.png
│   ├── ssd_car.jpg
│   ├── sysdump.mp4
│   ├── transcode_ppl.png
│   ├── ubuntu_apt_source.png
│   ├── uos_develop_mode.png
│   ├── uos_off_sign.png
│   ├── voc_dog.jpg
│   ├── voc_dog_yolov5s_out.jpg
│   ├── voc_horse.jpg
│   ├── voc_horse_yolov5s_out.jpg
│   ├── vs2022_install.png
│   ├── yolo11_out.jpg
│   ├── yolo11_pose_out.jpg
│   ├── yolo11_seg_out.jpg
│   ├── yolo_world_out.jpg
│   └── yolov7_face_out.jpg
└── source
    ├── axcl_error_lookup.html
    ├── conf.py
    ├── doc_guide_axcl_api.md
    ├── doc_guide_axcl_smi.md
    ├── doc_guide_compile.md
    ├── doc_guide_faq.md
    ├── doc_guide_ffmpeg.md
    ├── doc_guide_hardware.md
    ├── doc_guide_npu_benchmark.md
    ├── doc_guide_npu_samples.md
    ├── doc_guide_quick_start.md
    ├── doc_guide_samples.md
    ├── doc_guide_setup.md
    ├── doc_guide_win_setup.md
    ├── doc_introduction.md
    ├── doc_update_info.md
    └── index.rst
```

### 2.2 Build

Install dependencies:

```bash
pip install -r requirements.txt
```

Execute the following command in the project root directory:

```bash
$ make clean
$ make html
```

### 2.3 Local Preview

After building, use a browser to view `build/html/index.html`.

## 3. Reference

This project is based on Sphinx. More information about Sphinx can be found here: https://www.sphinx-doc.org/en/master/

## 4. Online Publishing

Online web service hosted on [ReadtheDocs](https://readthedocs.org/) platform.

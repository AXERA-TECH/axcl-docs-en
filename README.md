# AXCL User Manual

[Web Preview]()

## 1. Project background

**AXCL** is the Host API for AX650N-based PCIE EP products.
E
- Provide related APIs and Examples of AI computing use with AXCL.
- Facilitate community developers to co-create documents.

## 2. Local operation guide

### 2.1 git clone

```bash
git clone https://github.com/AXERA-TECH/axcl-docs.git
```

The directory tree is as follows:

```bash
.
├── LICENSE
├── Makefile
├── README.md
├── requirements.txt
└── source
    ├── conf.py
    ├── doc_guide_axcl_api.md
    ├── doc_guide_hardware.md
    ├── doc_guide_quick_start.md
    ├── doc_introduction.md
    ├── doc_update_info.md
    ├── index.rst
    └── media
        ├── demo
        └── hardware
```

### 2.2 Compiled

Installation dependency

```bash
pip install -r requirements.txt
```

Execute the following command in the project root directory

```bash
$ make clean
$ make html
```

### 2.3 Preview

After compiling, use a browser to view it `build/html/index.html`

## 3. Refer to

This project is based on Sphinx, more information about Sphinx can be found here https://www.sphinx-doc.org/en/master/

## 4. Release

Online proxy based on [ReadtheDocs](https://readthedocs.org/) platform.


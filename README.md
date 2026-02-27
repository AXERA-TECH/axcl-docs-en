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
├── requirements.txt
└── source
    ├── conf.py
    ├── doc_guide_axcl_api.md
    ├── doc_guide_faq.md
    ├── doc_guide_hardware.md
    ├── doc_guide_quick_start.md
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

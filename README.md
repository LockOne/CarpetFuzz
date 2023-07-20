# CarpetFuzz #

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.8166412.svg)](https://doi.org/10.5281/zenodo.8166412)

<p><a href="https://www.usenix.org/system/files/sec23fall-prepub-467-wang-dawei.pdf"><img alt="CarpetFuzz thumbnail" align="right" width="200" src="images/CarpetFuzz-thumbnail.png"></a></p>

CarpetFuzz is an NLP-based fuzzing assistance tool for generating valid option combinations.

The basic idea of CarpetFuzz is to use natural language processing (NLP) to identify and extract the relationships (e.g., conflicts or dependencies) among program options from the description of each option in the documentation and filter out invalid combinations to reduce the option combinations that need to be fuzzed.

For more details, please refer to [our paper](https://www.usenix.org/system/files/sec23fall-prepub-467-wang-dawei.pdf) from USENIX Security'23.

The [CarpetFuzz-experiments](https://github.com/waugustus/CarpetFuzz-experiments) repository contains the data sets, scripts, and documentation required to reproduce our results in the paper.

## Structure ##

|Directory|Description|
|---|---|
|[dataset](dataset)|Training dataset to obtain the model|
|[fuzzer](https://github.com/waugustus/CarpetFuzz-fuzzer)|Modified fuzzer to switch combinations on the fly. (Submodule)|
|[images](images)|Images used in README.md| 
|[models](models)|Models used to extract relationships|
|[output](output)|CarpetFuzz's output files|
|[pict](https://github.com/microsoft/pict)|Microsoft's pairwise tool. (Submodule)|
|[scripts](scripts)|Python scripts to identify, extract relationships and rank combinations based on their dry-run coverage.|
|[scripts/utils](scripts/utils)|Some general purposed utility class.|
|[tests](tests)|Some sample files to test CarpetFuzz.|
|[tests/dict](tests/dict)|Sample dictionary file used to generate stub (involving 49 programs).|
|[tests/manpages](tests/manpages)|Sample manpage files.|

We have highly structured our code and provided extensive comments to enhance readers' comprehension. The implementations for various components of CarpetFuzz can be found in the following functions,

| Section | Component | File | Function |
|----|----|----|----|
| 3.2 | EDR Identification | [scripts/find_relationship.py](scripts/find_relationship.py) | identifyExplicitRSentences |
| 3.3 | IDR Identification | [scripts/find_relationship.py](scripts/find_relationship.py) | identifyImplicitRSentences |
| 3.4 | Relationship Extraction | [scripts/find_relationship.py](scripts/find_relationship.py) | extractRelationships |
| 3.5 | Combination | [scripts/generate_combination.py](scripts/generate_combination.py) | main |
| 3.5 | Prioritization | [scripts/rank_combination.py](scripts/rank_combination.py) | main |

## Supported Environments ##

CarpetFuzz is recommended to be run on Linux systems. We have tested it on the following operating system versions:

- Ubuntu 18.04
- Ubuntu 20.04

While our testing has primarily focused on these operating systems, it may also work on other Linux distributions. Ensure that your system meets the following requirements:

- Linux operating system (Ubuntu 18.04 or 20.04 is recommended)
- Python 3.6 or higher
- LLVM 12.0.0 or higher
- Required dependencies (detailed instructions will be provided during the installation process)

Please note that CarpetFuzz may not be compatible with non-Linux systems or may encounter compatibility issues on other operating systems. We recommend running CarpetFuzz on a supported Linux distribution for the best user experience and performance.

## Installation ##

For easy installation, we offer a ready-to-use Docker image for download,

```
sudo docker pull 4ugustus/carpetfuzz 
```

or you can compile the image yourself using the Dockerfile we provide.

```
# Download CarpetFuzz repo with the submodules
git clone --recursive https://github.com/waugustus/CarpetFuzz
cd CarpetFuzz
# Build image
sudo docker build -t carpetfuzz:latest .
```

And you can also build CarpetFuzz yourself:

```
# Download CarpetFuzz repo with the submodules
git clone --recursive https://github.com/waugustus/CarpetFuzz
cd CarpetFuzz

# Build CarpetFuzz-fuzzer (LLVM 11.0+ is recommended)
pushd fuzzer
make clean all
popd

# Build Microsoft pict
pushd pict
cmake -DCMAKE_BUILD_TYPE=Release -S . -B build
cmake --build build
pushd build && ctest -v && popd
popd

# Install required pip modules (virtualenv is recommended)
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
python3 -m spacy download en_core_web_sm-3.0.0 --direct
echo -e "import nltk\nnltk.download('averaged_perceptron_tagger')\nnltk.download('omw-1.4')\nnltk.download('punkt')\nnltk.download('wordnet')"|python3

# Download AllenNLP's parser model
wget -P models/ https://allennlp.s3.amazonaws.com/models/elmo-constituency-parser-2020.02.10.tar.gz
```


## Usage (Minimal Working Example) ##

We take the program `tiffcp` used in the paper as an example,

```
export CarpetFuzz=/path/to/CarpetFuzz

# Step 1
# Download and build the tiffcp repo with CarpetFuzz-fuzzer
git clone https://gitlab.com/libtiff/libtiff
cd libtiff
git reset --hard b51bb
sh ./autogen.sh
CC=${CarpetFuzz}/fuzzer/afl-clang-fast CXX=${CarpetFuzz}/fuzzer/afl-clang-fast++ ./configure --prefix=$PWD/build_carpetfuzz --disable-shared
make -j;make install;make clean
# Prepare the seed
mkdir input
cp ${CarpetFuzz}/fuzzer/testcases/images/tiff/* input/

# Step 2
# Use CarpetFuzz to analyze the relationships from the manpage file
python3 ${CarpetFuzz}/scripts/find_relationship.py --file $PWD/build_carpetfuzz/share/man/man1/tiffcp.1
# Based on the relationship, use pict to generate 6-wise combinations
python3 ${CarpetFuzz}/scripts/generate_combination.py --relation ${CarpetFuzz}/output/relation/relation_tiffcp.json
# Rank each combination with its dry-run coverage
python3 ${CarpetFuzz}/scripts/rank_combination.py --combination ${CarpetFuzz}/output/combination/combination_tiffcp.txt --dict ${CarpetFuzz}/tests/dict/dict.json --bindir $PWD/build_carpetfuzz/bin --seeddir input

# Step 3
# Fuzz with the ranked stubs
${CarpetFuzz}/fuzzer/afl-fuzz -i input/ -o output/ -K ${CarpetFuzz}/output/stubs/ranked_stubs_tiffcp.txt -- $PWD/build_carpetfuzz/bin/tiffcp @@
```

## FAQ ##

1. How to find the manpage file of a new program?
   
   In our experience, manpage files are typically located in the `share` directory within the compilation directory, such as `/your_build_dir/share/man/man1`.

2. How to know which option combination triggered a crash?
   
   You can extract the corresponding argv index from the crash filename. For instance, the filename `id:000000,sig:07, src:000000,argv:000334,op:argv,pos:0` indicates that the crash was triggered by `argv:000334`. You can then find the corresponding argv in `line 336` (i.e., 334+2) of the ranked_stubs file.

3. How to reduce memory consumption when using pict to combine a large number of options?
   
   When there is a large number of options (e.g., `gm`), PICT consumes a significant amount of memory (more than 128GB). In such cases, you can restricted the number of options by sorting all individual options based on their dry-run coverage and selecting the top 50 options with the highest coverage for combination. The whole process can be done by the `simplify_relationship.py` script.

    ```
    # Restrict the number of options based on their coverage
    python3 ${CarpetFuzz}/scripts/simplify_relation.py --relation ${CarpetFuzz}/output/relation/relation_gm.json --dict ${CarpetFuzz}/tests/dict/dict.json --bindir $PWD/build_carpetfuzz/bin --seeddir input
    ```

## CVEs found by CarpetFuzz ##

CarpetFuzz has found 56 crashes on our real-world dataset, of which 42 are 0-days. So far, 20 crashes have been assigned with CVE IDs.

|CVE|Program|Type|
|---|---|---|
|CVE-2022-0865|tiffcp|assertion failure|
|CVE-2022-0907|tiffcrop|segmentation violation|
|CVE-2022-0909|tiffcrop|floating point exception|
|CVE-2022-0924|tiffcp|heap buffer overflow|
|CVE-2022-1056|tiffcrop|heap buffer overflow|
|CVE-2022-1622|tiffcp|segmentation violation|
|CVE-2022-1623|tiffcp|segmentation violation|
|CVE-2022-2056|tiffcrop|floating point exception|
|CVE-2022-2057|tiffcrop|floating point exception|
|CVE-2022-2058|tiffcrop|floating point exception|
|CVE-2022-2953|tiffcrop|heap buffer overflow|
|CVE-2022-3597|tiffcrop|heap buffer overflow|
|CVE-2022-3598|tiffcrop|heap buffer overflow|
|CVE-2022-3599|tiffcrop|heap buffer overflow|
|CVE-2022-3626|tiffcrop|heap buffer overflow|
|CVE-2022-3627|tiffcrop|heap buffer overflow|
|CVE-2022-4450|openssl-asn1parse|double free|
|CVE-2022-4645|tiffcp|heap buffer overflow|
|CVE-2022-29977|img2sixel|assertion failure|
|CVE-2022-29978|img2sixel|floating point exception|
|CVE-2023-0795|tiffcrop|segmentation violation|
|CVE-2023-0796|tiffcrop|segmentation violation|
|CVE-2023-0797|tiffcrop|segmentation violation|
|CVE-2023-0798|tiffcrop|segmentation violation|
|CVE-2023-0799|tiffcrop|heap use after free|
|CVE-2023-0800|tiffcrop|heap buffer overflow|
|CVE-2023-0801|tiffcrop|heap buffer overflow|
|CVE-2023-0802|tiffcrop|heap buffer overflow|
|CVE-2023-0803|tiffcrop|heap buffer overflow|
|CVE-2023-0804|tiffcrop|heap buffer overflow|

## Credit ##

Thanks to Ying Li ([@Fr3ya](https://github.com/Fr3ya)) and Zhiyu Zhang ([@QGrain](https://github.com/QGrain)) for their valuable contributions to this project.

## Citing this paper ##

In case you would like to cite CarpetFuzz, you may use the following BibTex entry:

```
@inproceedings {
  title = {CarpetFuzz: Automatic Program Option Constraint Extraction from Documentation for Fuzzing},
  author = {Wang, Dawei and Li, Ying and Zhang, Zhiyu and Chen, Kai},
  booktitle = {Proceedings of the 32nd USENIX Conference on Security Symposium},
  publisher = {USENIX Association},
  address = {Anaheim, CA, USA},
  pages = {},
  year = {2023}
}
```

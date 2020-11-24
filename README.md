### Introduction

This workflow is written using snakemake language with some python language inside, it can be used to proteomics research,especially for DIA MS collected data. It allows users to generate library and get EncyclopeDIA quantitative results.

In order to execute snakemake workflow, the installation of java,python,docker and snakemake is required. The absolute path of the files to generate the library and to get quantitative results should be pass to the workflow as variable values via the command line.



### Environment and software supports

java
docker
encyclopedia-0.9.5-executable.jar
python
snakemake



### Setting of work directory

```bash
.
├── bin
│   └── encyclopedia-0.9.5-executable.jar
├── ref
│   ├── uniprot_human_25apr2019.fasta
│   └── uniprot_human_25apr2019.fasta.z2_nce33.dlib
└── snakefile
```

encyclopedia-0.9.5-executable.jar can be downloaded at https://bitbucket.org/searleb/encyclopedia/downloads/encyclopedia-0.9.5-executable.jar; uniprot_human_25apr2019.fasta can be downloaded from https://www.proteomicsdb.org/prosit/api/libraries.xsjs?file=uniprot_human_25apr2019.fasta; uniprot_human_25apr2019.fasta.z2_nce33.dlib can be downloaded at https://www.proteomicsdb.org/prosit/api/libraries.xsjs?file=uniprot_human_25apr2019.fasta.z2_nce33.dlib.



### Uses

```bash
$Library_dir='your path of files to generate the library' Quantitation_dir='your path of files to generate quantitative reports' snakemake
```

In this project, data and directory setting is:

```bash
/mnt/external/dia_data/s-trap
├── Library_Generation
│   ├── 190529_Plasma_S-Trap_DIA_Window400_500_01.raw
│   ├── 190529_Plasma_S-Trap_DIA_Window500_600_01.raw
│   ├── 190529_Plasma_S-Trap_DIA_Window600_700_01.raw
│   ├── 190529_Plasma_S-Trap_DIA_Window700_800_01.raw
│   ├── 190529_Plasma_S-Trap_DIA_Window800_900_01.raw
│   └── 190529_Plasma_S-Trap_DIA_Window900_1000_01.raw
└── Quantitative_results
    ├── 190529_Plasma_S-Trap_DIA_FullWindow_01.raw
    ├── 190529_Plasma_S-Trap_DIA_FullWindow_02.raw
    └── 190529_Plasma_S-Trap_DIA_FullWindow_03.raw
```

After executing this command below, "uniprot_human_25apr2019.elib" is generated inside "Library_Generation" directory and quantitative reports are generated inside "Quantitative_results" directory.

```bash
$Library_dir="/mnt/external/dia_data/s-trap/Library_Generation/" Quantitation_dir="/mnt/external/dia_data/s-trap/Quantitative_results" sankemake
```



### snakefile:

```bash
# Get library generation directory and quantitation directory
lib_dir =  os.environ.get("Library_dir")
quant_dir = os.environ.get("Quantitation_dir")

# Define a function used to generate library 
def file_name(file_dir):
    ids=[]
    for root, dirs, files in os.walk(file_dir):
        for file in files:
            if os.path.splitext(file)[1] == '.raw':
                ids.append(file.split('.')[0])
    return ids

# Get file name
lib_ids = file_name(lib_dir)
quant_ids = file_name(quant_dir)

# Output quantitative reports
rule generate_quantitative_reports:
    params:
        input_mzML_dir = quant_dir,
        input_fasta_file = "ref/uniprot_human_25apr2019.fasta",
        input_dlib_file = "ref/uniprot_human_25apr2019.fasta.z2_nce33.dlib",
        input_software = "bin/encyclopedia-0.9.5-executable.jar",
        output_files = "Encyclopedia"
    input:
        lib_dir+"uniprot_human_25apr2019.elib"
    output:
        quant_dir+"Encyclopedia.peptides.txt",
        quant_dir+"Encyclopedia.proteins.txt"
    shell:
        """
        cd {params.input_mzML_dir};
        java -Xmx12g -jar {params.input_software} -i {params.input_mzML_dir} -l {input} -f {params.input_fasta_file};
        java -Xmx12g -jar {params.input_software} -libexport -i {params.input_mzML_dir} -l {input} -f {params.input_fasta_file} -o {params.output_files} -a true
        """

# Transfer raw file used to quantitative analysis to mzML format
rule quantitation_raw_to_mzML:
    params:
        input_dir = quant_dir
    input:
        expand(quant_dir+"{id}.raw",id=quant_ids)
    output:
        expand(quant_dir+"{id}.mzML",id=quant_ids)
    shell:
        """
        cd {params.input_dir};
        sudo docker run -it --rm -e WINEDEBUG=-all -v {params.input_dir}:/data chambm/pwiz-skyline-i-agree-to-the-vendor-licenses wine msconvert --mzML *.raw --filter="peakPicking true 1-q" --filter="demultiplex massError=10ppm optimization=overlap_only"
        """


# Generate library
rule generate_elib_file:
    params:
        input_mzML_dir = lib_dir,
        input_fasta_file = "ref/uniprot_human_25apr2019.fasta",
        input_dlib_file = "ref/uniprot_human_25apr2019.fasta.z2_nce33.dlib",
        input_software = "bin/encyclopedia-0.9.5-executable.jar"
    input:
        input_mzML_files = expand(lib_dir+"{id}.mzML",id=lib_ids)
    output:
        lib_dir+"uniprot_human_25apr2019.elib"
    shell:
        """
        cd {params.input_mzML_dir};
        java -Xmx12g -jar {params.input_software} -i {params.input_mzML_dir} -l {params.input_dlib_file} -f {params.input_fasta_file};
        java -Xmx12g -jar {params.input_software} -libexport -i {params.input_mzML_dir} -l {params.input_dlib_file} -f {params.input_fasta_file} -o {output} -a false
        """

# Convert raw file used to generate library to mzML format
rule library_raw_to_mzML:
    params:
        input_dir = lib_dir
    input:
        expand(lib_dir+"{id}.raw",id=lib_ids)
    output:
        expand(lib_dir+"{id}.mzML",id=lib_ids)
    shell:
        """
        cd {params.input_dir};
        sudo docker run -it --rm -e WINEDEBUG=-all -v {params.input_dir}:/data chambm/pwiz-skyline-i-agree-to-the-vendor-licenses wine msconvert --mzML *.raw --filter="peakPicking true 1-q" --filter="demultiplex massError=10ppm optimization=overlap_only"
        """
```




### Author
Lina Lu
Department of Biology, Box 118, 221 00, Lund University, Sweden

20/11/2020
# Assignment 4: De Novo Genome Assembly Pipeline

## Overview
In this assignment, you will build a **Snakemake workflow** for de novo genome assembly. Unlike previous assignments where specific tools were prescribed, this assignment gives you flexibility to choose appropriate assemblers and quality control tools based on your sequencing data type (short-read, long-read, or hybrid assembly). You will assemble a genome, assess its quality, and annotate basic assembly statistics.

## Learning Objectives
By completing this assignment, you will:
- Design and implement a complete de novo assembly pipeline
- Select appropriate assembly tools based on data characteristics
- Assess genome assembly quality using multiple metrics
- Generate comprehensive assembly statistics and visualizations
- Practice decision-making in bioinformatics workflow design
- Manage large-scale computational workflows with Snakemake

## Prerequisites
Before starting this assignment, ensure you have:
- Python 3.8+
- [Snakemake](https://snakemake.readthedocs.io/) (v7.0+)
- Conda/Mamba for package management

The specific assembly and analysis tools you need will depend on your data type. Consider tools from these categories:

### Short-read assemblers (Illumina):
- SPAdes
- MEGAHIT
- ABySS
- Velvet

### Long-read assemblers (PacBio/Nanopore):
- Flye
- Canu
- Hifiasm (for PacBio HiFi)
- Raven
- NextDenovo/NextPolish

### Hybrid assemblers (short + long reads):
- SPAdes (hybrid mode)
- MaSuRCA
- Unicycler

### Quality assessment tools:
- QUAST (assembly statistics)
- BUSCO (completeness assessment)
- Bandage (assembly graph visualization)
- Merqury or KAT (k-mer based QC)
- Minimap2 + samtools (for coverage analysis)

### Data preprocessing tools:
- FastQC / NanoPlot / LongQC (quality control)
- Fastp / Trimmomatic (short-read trimming)
- Filtlong / NanoFilt (long-read filtering)
- MultiQC (aggregate reports)

**Important:** For this assignment, you must use **per-rule conda or apptainer environments**. This ensures reproducibility and proper dependency management. Do not use a single global environment.

Create environment YAML files in `workflow/envs/` for each rule. Example:

```bash
# Create environment files for your tools
# Example: workflow/envs/fastqc.yaml
# Example: workflow/envs/spades.yaml
# Example: workflow/envs/quast.yaml
```

## Data
For this assignment, you must provide your own sequencing data. You have several options:

### Option 1: Use Your Research Data
If you have sequencing data from your own research:
- Organize raw reads in `data/raw_reads/`
- Document the organism, sequencing platform, and coverage in your report
- Ensure you have permission to use the data

### Option 2: Download Public Data
Download datasets from public repositories:

**Short-read data (SRA/ENA):**
```bash
# Install SRA toolkit
mamba install -c bioconda sra-tools

# Download paired-end Illumina data
prefetch SRR_ID
fasterq-dump --split-files --gzip SRR_ID -O data/raw_reads/
```

**Long-read data (SRA/ENA):**
```bash
# Download PacBio or Nanopore data
prefetch SRR_ID
fasterq-dump --gzip SRR_ID -O data/raw_reads/
```

**Hybrid data:**
- Obtain both short and long reads for the same organism
- Organize in separate subdirectories (e.g., `data/raw_reads/illumina/` and `data/raw_reads/nanopore/`)

### Recommended Test Organisms
For testing your workflow, consider these smaller genomes:
- **Bacteria** (~1-10 Mb): E. coli, Salmonella, Mycobacterium
- **Yeast** (~12 Mb): Saccharomyces cerevisiae

You may also use a subset of reads from a larger genome for testing purposes.

**Important:** Start with a smaller genome or subset to test your workflow before scaling up!

### Data Organization
Organize your data as:
```
data/
├── raw_reads/
│   ├── sample_R1.fastq.gz  # Short reads
│   ├── sample_R2.fastq.gz  # Short reads
│   └── nanopore.fastq.gz   # Or long reads
└── metadata.txt            # Document your data source
```

## Getting Started

Follow these steps to set up and complete your genome assembly workflow.

### Step 1: Understand Your Data

Before assembling, you need to know:
- [ ] What sequencing platform? (Illumina, PacBio, Nanopore)
- [ ] Short reads, long reads, or hybrid?
- [ ] Paired-end or single-end (for short reads)?
- [ ] What is the expected genome size?
- [ ] What is the estimated coverage?

**Calculate coverage:**
```
Coverage = (Number of reads × Read length) / Genome size

Example:
- 5 million paired reads (2 × 150 bp)
- Expected genome size: 5 Mb
- Coverage = (5M × 300) / 5M = 300x
```

### Step 2: Document Your Data

Fill out `data/metadata.txt` with:
- Organism name
- Data source (SRA accession or your own)
- Sequencing platform and details
- Expected genome size
- Estimated coverage

### Step 3: Configure Your Workflow

Edit `config/config.yaml`:

1. **Update sample name:**
   ```yaml
   sample_name: "my_ecoli"
   ```

2. **Set data paths:**
   ```yaml
   # For paired-end short reads:
   reads_r1: "data/raw_reads/sample_R1.fastq.gz"
   reads_r2: "data/raw_reads/sample_R2.fastq.gz"
   ```

3. **Set organism info:**
   ```yaml
   organism: "Escherichia coli K-12"
   expected_genome_size: "4.6m"
   ```

4. **Choose BUSCO lineage:**
   ```yaml
   busco_lineage: "bacteria_odb10"
   # Other options: fungi_odb10, metazoa_odb10, embryophyta_odb10
   # See https://busco.ezlab.org/ for full list
   ```

5. **Set computational resources:**
   ```yaml
   assembly:
     threads: 8  # Adjust based on your system
     memory_gb: 16
   ```

### Step 4: Plan Your Workflow

Decide which tools to use based on your data:

**For Illumina Short Reads:**
1. **QC**: FastQC + MultiQC
2. **Preprocessing**: fastp or Trimmomatic
3. **Assembly**: SPAdes, MEGAHIT, or ABySS
4. **Quality**: QUAST + BUSCO

**For PacBio/Nanopore Long Reads:**
1. **QC**: NanoPlot or LongQC
2. **Preprocessing**: Filtlong (optional, some assemblers do this)
3. **Assembly**: Flye, Canu, or Hifiasm (for PacBio HiFi)
4. **Polishing**: Racon/Medaka (for Nanopore) or Arrow (for PacBio)
5. **Quality**: QUAST + BUSCO + minimap2 coverage

**For Hybrid (Short + Long):**
1. **QC**: FastQC for short, NanoPlot for long
2. **Assembly**: SPAdes hybrid mode or Unicycler
3. **Quality**: QUAST + BUSCO

### Step 5: Create Environment Files

**Required:** Create conda environment YAML files in `workflow/envs/` for each tool you use.

**Why per-rule environments?**
- Ensures **reproducibility** - Anyone can recreate your exact software environment
- Prevents **dependency conflicts** - Different tools may require different versions of libraries
- Enables **parallel installation** - Snakemake creates environments as needed
- Documents **exact versions** - Makes it clear what software was used

**Environment File Structure:**
Each YAML file should specify:
1. **Channels** - Repositories to search for packages (bioconda, conda-forge)
2. **Dependencies** - Tools and their exact versions

**Example:** `workflow/envs/fastqc.yaml`
```yaml
channels:
  - bioconda
  - conda-forge
dependencies:
  - fastqc=0.12.1
  - multiqc=1.14
```

Create similar files for other tools (e.g., `spades.yaml`, `flye.yaml`, `quast.yaml`, `busco.yaml`)

**Finding Package Versions:**
```bash
# Search for available versions
conda search -c bioconda fastqc

# Or visit https://bioconda.github.io/
```

**Creating and Exporting Environments:**
You can create an environment interactively, test it, then export to YAML:

```bash
# Create a new environment and install packages
conda create -n test_fastqc -c bioconda -c conda-forge fastqc=0.12.1 multiqc=1.14

# Activate and test the tools
conda activate test_fastqc
fastqc --version

# Export to YAML file
conda env export --from-history > workflow/envs/fastqc.yaml

# Clean up the file - remove any unnecessary lines (like prefix, name)
# Keep only: channels and dependencies sections

# Deactivate when done
conda deactivate

# Remove test environment
conda env remove -n test_fastqc
```

**Note:** Using `--from-history` exports only packages you explicitly installed, not all dependencies. This creates cleaner, more portable YAML files.

**Testing Environments (optional but recommended):**
```bash
# Test an environment before adding to workflow
conda env create -f workflow/envs/fastqc.yaml -n test_fastqc
conda activate test_fastqc
fastqc --version
conda deactivate
```

### Step 6: Edit the Snakefile

Open `workflow/Snakefile` and:

1. **Add rules** for your data type
2. **Fill in input/output paths**
3. **Add shell commands** for your chosen tools
4. **Add conda or apptainer directive to EVERY rule**
5. **Update the `rule all`** with your expected outputs

Example of implementing a FastQC rule with conda:
```python
rule raw_qc:
    input:
        r1 = config["reads_r1"],
        r2 = config["reads_r2"]
    output:
        html1 = f"{QC_DIR}/raw/{SAMPLE}_R1_fastqc.html",
        html2 = f"{QC_DIR}/raw/{SAMPLE}_R2_fastqc.html"
    params:
        outdir = f"{QC_DIR}/raw"
    threads: 2
    log:
        f"{LOG_DIR}/fastqc_{SAMPLE}.log"
    conda:
        "envs/fastqc.yaml"  # Required!
    shell:
        """
        mkdir -p {params.outdir}
        fastqc {input.r1} {input.r2} -o {params.outdir} -t {threads} &> {log}
        """
```

**Alternative with apptainer:**
```python
rule raw_qc:
    input:
        r1 = config["reads_r1"],
        r2 = config["reads_r2"]
    output:
        html1 = f"{QC_DIR}/raw/{SAMPLE}_R1_fastqc.html",
        html2 = f"{QC_DIR}/raw/{SAMPLE}_R2_fastqc.html"
    params:
        outdir = f"{QC_DIR}/raw"
    threads: 2
    log:
        f"{LOG_DIR}/fastqc_{SAMPLE}.log"
    container:
        "docker://biocontainers/fastqc:v0.11.9_cv8"  # Required!
    shell:
        """
        mkdir -p {params.outdir}
        fastqc {input.r1} {input.r2} -o {params.outdir} -t {threads} &> {log}
        """
```

### Step 7: Test Your Workflow

**Start with a dry-run:**
```bash
snakemake --dry-run --cores 1
```
This checks for errors without running anything.

**Run the full workflow with conda:**
```bash
snakemake --use-conda --cores 8
```

**Or with apptainer (if using container directive):**
```bash
snakemake --use-singularity --cores 8
```

**Or with apptainer and conda (if using both directives):**
```bash
snakemake --use-singularity --use-conda --cores 8
```

### Step 8: Monitor Progress

**Check log files:**
```bash
# View logs in real-time
tail -f logs/assembly_my_sample.log
```

**Check intermediate outputs:**
```bash
# Look at QC reports
ls -lh results/qc/
```

### Step 9: Evaluate Results

After assembly completes:

1. **Check QUAST report:**
   ```bash
   # Open in browser
   open results/qc/assembly/quast/report.html
   ```

2. **Review BUSCO results:**
   ```bash
   cat results/qc/assembly/busco/short_summary.txt
   ```

3. **Create your report:**
   - Fill in `report.md` with your results
   - Include specific metrics and interpretations

## Assignment Tasks

### Task 1: Data Quality Control
Implement Snakemake rules to:
1. Assess raw read quality using appropriate tools for your data type
2. Generate summary statistics (read length distribution, quality scores, GC content)
3. Create visualizations of read quality
4. Aggregate QC reports if using multiple tools

#### Expected outputs:
- `results/qc/raw/{sample}_qc_report.html` (or similar)
- `results/qc/multiqc_report.html` (if applicable)

### Task 2: Read Preprocessing (if necessary)
Implement Snakemake rules to:
1. Filter and/or trim reads based on quality
2. Remove adapters (for short reads)
3. Filter by length (for long reads)
4. Re-assess quality after preprocessing
5. **Document your filtering criteria and rationale**

#### Expected outputs:
- `results/processed_reads/{sample}_filtered.fastq.gz`
- `results/qc/processed/{sample}_qc_report.html`

**Note:** Some assemblers perform their own filtering. Document whether you do preprocessing or rely on the assembler.

### Task 3: Genome Assembly
Implement Snakemake rules to:
1. Run your chosen assembler with appropriate parameters
2. Document parameter choices in comments or config file
3. Consider testing multiple parameter sets if computationally feasible
4. Save assembly graphs (if available from assembler)

#### Expected outputs:
- `results/assembly/contigs.fasta` (or scaffolds.fasta)
- `results/assembly/assembly_graph.gfa` (if available)

**Key considerations:**
- Choose k-mer size (for short reads)
- Set coverage thresholds
- Specify expected genome size (if known)
- Document all non-default parameters

### Task 4: Assembly Quality Assessment
Implement Snakemake rules to assess assembly quality using multiple approaches:

#### 4a. Basic Assembly Statistics (using QUAST or similar)
Generate statistics including:
- Number of contigs/scaffolds
- Total assembly length
- N50, N75, N90
- Largest contig size
- GC content

#### 4b. Completeness Assessment (using BUSCO)
- Choose appropriate lineage dataset for your organism
- Assess completeness using single-copy orthologs
- Document BUSCO scores (Complete, Fragmented, Missing)

#### 4c. Additional Quality Metrics (choose at least one)
- **K-mer analysis**: Compare assembled k-mers to read k-mers (Merqury/KAT)
- **Read mapping**: Map raw reads back to assembly (coverage uniformity)
- **Assembly graph visualization**: Visualize with Bandage
- **Contamination screening**: Check for contaminants (if applicable)

#### Expected outputs:
- `results/qc/assembly/quast_report.html`
- `results/qc/assembly/busco_summary.txt`
- `results/qc/assembly/` (additional QC outputs)

### Task 5: Assembly Polishing (Optional but Recommended)
For long-read assemblies, implement polishing rules:
1. Polish with long reads (Racon, Medaka for Nanopore)
2. Polish with short reads if available (Pilon, Polypolish)
3. Compare assembly quality before and after polishing

#### Expected outputs (if implemented):
- `results/polished/polished_assembly.fasta`
- `results/qc/polished/` (QC for polished assembly)

### Task 6: Workflow Configuration
Create a proper Snakemake workflow structure:
1. Use a `Snakefile` with well-documented rules
2. **Add `conda:` or `container:` directive to EVERY rule** (required)
3. Create environment YAML files in `workflow/envs/` for each tool
4. Create a `config.yaml` file for:
   - Data paths
   - Tool parameters
   - Expected genome size
   - BUSCO lineage
   - Number of threads/cores
5. Use wildcards and functions where appropriate
6. Define an `all` rule that specifies all final outputs
7. Use `log:` directives to capture tool outputs
8. Add extensive comments explaining your choices

### Task 7: Visualization and Reporting
1. Generate a workflow diagram (DAG)
2. If using assembly graphs, create Bandage visualizations
3. Create plots for:
   - Contig length distribution
   - Coverage distribution (if calculated)
   - BUSCO results visualization

#### Expected outputs:
- `dag.png` - Workflow visualization
- `results/plots/contig_length_distribution.pdf`
- `results/plots/busco_plot.png` (if using BUSCO plotting script)

### Task 8: Documentation and Analysis Report
Create a comprehensive report (`report.md`) including:

1. **Introduction** (1 paragraph):
   - Organism being assembled
   - Data source and sequencing platform
   - Rationale for tool choices

2. **Methods** (2-3 paragraphs):
   - Assembly approach (short-read, long-read, or hybrid)
   - Tools selected and why
   - Key parameters and their justification
   - Preprocessing steps performed

3. **Results** (2-3 paragraphs):
   - Raw data quality summary
   - Assembly statistics (N50, total length, contig count)
   - BUSCO completeness scores
   - Other QC metrics
   - Comparison to expected genome size (if known)

4. **Discussion** (1-2 paragraphs):
   - Quality assessment of the assembly
   - Potential improvements or next steps
   - Challenges encountered and how you addressed them
   - Limitations of the assembly

5. **References**:
   - Cite all tools used
   - Data source citations

## Workflow Structure
Your final directory structure should look like:
```
.
├── workflow/
│   ├── Snakefile
│   ├── envs/            # Required: per-rule conda/container envs
│   │   └── *.yaml
│   └── scripts/         # Any custom scripts
│       └── *.py or *.R
├── config/
│   └── config.yaml
├── data/
│   ├── raw_reads/
│   │   └── *.fastq.gz
│   └── metadata.txt
├── results/
│   ├── qc/
│   │   ├── raw/
│   │   ├── processed/
│   │   └── assembly/
│   ├── processed_reads/
│   ├── assembly/
│   ├── polished/        # If applicable
│   └── plots/
├── logs/
├── report.md
└── dag.png
```

## Deliverables
Submit the following via GitHub by merging your working branch into the main branch via a pull request:

**Required:**
1. `workflow/Snakefile` - Your complete workflow with extensive comments
2. `workflow/envs/*.yaml` - Conda environment files for each tool (required)
3. `config/config.yaml` - Well-documented configuration file
4. `report.md` - Comprehensive analysis report following the outline above
5. `dag.png` - DAG visualization of your workflow
6. `data/metadata.txt` - Documentation of your data source, organism, and sequencing details
7. `.gitignore` - Properly configured to exclude large files
8. Assembly QC outputs:
   - QUAST report
   - BUSCO summary
   - At least one additional QC metric

**Optional (but encouraged):**
9. `workflow/scripts/` - Any custom analysis scripts
10. `results/plots/` - Visualization outputs
11. Polishing results and comparisons

**Do not commit:**
- Large sequencing data files (FASTQ)
- Assembly FASTA files
- BAM/SAM files
- Intermediate files

Use `.gitignore` to exclude these automatically.

## Tips
- **Start small**: Test your workflow with a subset of reads before running the full assembly
- **Use `snakemake --dry-run`** frequently to check for errors
- **Always use `--use-conda`** when running your workflow: `snakemake --use-conda --cores 8`
- **Document everything**: Your future self will thank you
- **Read the manual**: Different assemblers have very different parameter requirements
- **Monitor resources**: Genome assembly can be memory and time intensive
- **Compare parameters**: If time allows, try different parameter sets and compare results
- **Use logging**: Direct stderr and stdout to log files for debugging
- **Modular design**: Write separate rules for each major step
- **Benchmark**: Use Snakemake's `benchmark:` directive to track resource usage
- **Per-rule environments are required**: Every rule must have a `conda:` or `container:` directive

## Common Pitfalls to Avoid
- **Forgetting `conda:` or `container:` directives** - Every rule must have one!
- **Not using `--use-conda`** when running Snakemake
- Placing environment files outside `workflow/envs/`
- Insufficient memory allocation for assembly
- Using inappropriate assemblers for your data type
- Not documenting parameter choices
- Skipping quality control steps
- Assembling without understanding your data characteristics
- Not checking for contamination in your reads
- Ignoring BUSCO lineage selection

## Resources

### Snakemake
- [Snakemake Documentation](https://snakemake.readthedocs.io/)
- [Snakemake Tutorial](https://snakemake.readthedocs.io/en/stable/tutorial/tutorial.html)
- [Using Conda Environments](https://snakemake.readthedocs.io/en/stable/snakefiles/deployment.html#integrated-package-management)
- [Using Containers](https://snakemake.readthedocs.io/en/stable/snakefiles/deployment.html#running-jobs-in-containers)
- [Snakemake Wrapper Repository](https://snakemake-wrappers.readthedocs.io/)

### Assembly Tools Documentation
**Short-read:**
- [SPAdes Manual](https://github.com/ablab/spades) - [Basic options](https://github.com/ablab/spades#basic-options)
- [MEGAHIT](https://github.com/voutcn/megahit)

**Long-read:**
- [Flye Documentation](https://github.com/fenderglass/Flye) - [Usage guide](https://github.com/fenderglass/Flye/blob/flye/docs/USAGE.md)
- [Canu Manual](https://canu.readthedocs.io/)
- [Hifiasm](https://github.com/chhylp123/hifiasm)

**Hybrid:**
- [Unicycler](https://github.com/rrwick/Unicycler) - [Basic usage](https://github.com/rrwick/Unicycler#basic-usage)
- [MaSuRCA](https://github.com/alekseyzimin/masurca)

## Due Date
**Sunday, March 29th at Midnight**

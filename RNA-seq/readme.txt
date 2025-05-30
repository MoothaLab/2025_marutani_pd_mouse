# This readme details the parameters used for QC and alignment of the RNA-seq
# sequencing reads. The reads were aligned to the mm10 mouse reference genome
# and the GENCODE vM24 genome annotation with STAR aligner version 2.7.5a.
# QC is carried out by FASTQC v0.11.9, with MultiQC v1.11 to compile the results for
# individual samples into a single report. The pipeline for processing the
# data is as reported in Rogers et al. 2023, and the code repository is
# public here: https://github.com/MoothaLab/ercc1-mouse-rob
#
# The fastq files were assumed to be named by the sample IDs, all stored in a
# single directory. The alignment pipeline is configured to write the results
# for each sample out to its own private directory.


#run FASTQC
cd ..;
mkdir fastqc_output;
for path in ./fastq/*.fastq.gz;
do
  qsub -cwd -j y -b y -N fastqc -o ./fastqc_output/ -l h_vmem=1g -l h_rt=4:00:00 -pe smp 4 -binding linear:4 "conda activate sc_remake2; fastqc -f fastq --outdir=./fastqc_output --threads=12 --quiet $path;";
done

mkdir fastqc_output/read1;
mkdir fastqc_output/read2;
mv fastqc_output/*_R1_001_fastqc* fastqc_output/read1/;
mv fastqc_output/*_R2_001_fastqc* fastqc_output/read2/;
multiqc --filename=read1_multiqc ./fastqc_output/read1/;
multiqc --filename=read2_multiqc ./fastqc_output/read2/;

#Align reads, including depleting reads aligning to rDNA
for fastq in ./fastq/*R1_001.fastq.gz;
do
  base_name=${fastq##*/};
  sample_name=${base_name%%_*};
  qsub -cwd -j y -b y -N bulk_rnaseq -o ./align_jobs_out/ -l h_rt=16:00:00 -l h_vmem=8G -pe smp 4 -binding linear:4 "conda activate sc_remake2; python bulk_rna-seq_pipeline.py $fastq ${fastq/R1/R2} --out_dir=./${sample_name} --aln_rdna_reference=refdata/mm10/star_index_rDNA_v2.7.5 --aln_reference=refdata/mm10/star_index_all_v2.7.5 --aln_reference_fasta=refdata/mm10/mm10_all_chrs.fa --aln_reference_sizes=refdata/mm10/mm10.chrom_sizes --aln_nthreads=4 --aln_annot_gtf=refdata/mm10/gencode.vM24.annotation.sorted.gtf --numts_bed=refdata/mm10/numts.blat.merged.bed --featureCounts_t=exon --max_concurrent_subjobs=50 --sample_name=${sample_name} --pipeline_jvmmemavail=30g --pipeline_recover;";
done;

#aggregate alignment statistics
multiqc --ignore ./fastqc_output --filename=alignment_multiqc --interactive ./;

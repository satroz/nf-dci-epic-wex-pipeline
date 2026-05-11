# nf-dci-epic-wex-pipeline
A pipeline for somatic variant calling in Whole Exome Sequencing

Tumor-Normal Whole Exome Sequencing (WES) Workflow for hg38

This document summarizes:

1. A full tumor-normal WES workflow for:
    * somatic SNV/indel calling
    * copy-number analysis
    * purity/ploidy estimation
    * contamination estimation
    * tumor-in-normal (TiN) estimation
2. A practical post-calling filtering and rescue workflow using:
    * Mutect2
    * DeepSomatic
    * CNVkit
    * deTiN

Reference genome: hg38 / GRCh38.

⸻

1. Full Tumor-Normal WES Workflow

Step	Tool(s)	Goal	Main Input Files	Main Output Files
1. FASTQ QC	FastQC, MultiQC	Assess sequencing quality, adapter content, GC bias, duplication, read counts	tumor_R1.fastq.gz, tumor_R2.fastq.gz, normal_R1.fastq.gz, normal_R2.fastq.gz	fastqc/*.html, fastqc/*.zip, multiqc_report.html
2. Alignment	BWA-MEM2	Align reads to hg38 reference genome	FASTQs, Homo_sapiens_assembly38.fasta, BWA index files	tumor.raw.bam, normal.raw.bam
3. Sort BAMs	samtools sort	Coordinate-sort alignments	*.raw.bam	tumor.sorted.bam, normal.sorted.bam
4. Mark duplicates	Picard MarkDuplicates or GATK MarkDuplicates	Identify PCR/optical duplicates	*.sorted.bam	tumor.markdup.bam, normal.markdup.bam, duplicate metrics
5. BAM indexing	samtools index	Create BAM indexes for downstream tools	*.markdup.bam	*.markdup.bam.bai
6. Base recalibration model (BQSR)	GATK BaseRecalibrator	Learn systematic sequencing error patterns	*.markdup.bam, Homo_sapiens_assembly38.fasta, dbsnp_146.hg38.vcf.gz, Mills_and_1000G_gold_standard.indels.hg38.vcf.gz, exome target BED	tumor.recal.table, normal.recal.table
7. Apply BQSR	GATK ApplyBQSR	Generate recalibrated BAMs	*.markdup.bam, recal tables	tumor.bqsr.bam, normal.bqsr.bam
8. BAM indexing after BQSR	samtools index	Index recalibrated BAMs	*.bqsr.bam	*.bqsr.bam.bai
9. Alignment/QC metrics	Picard CollectHsMetrics, Picard CollectInsertSizeMetrics, mosdepth, samtools stats	Evaluate mapping quality, insert size, exome capture efficiency, coverage	*.bqsr.bam, exome BED	HsMetrics, insert-size metrics, mapping metrics, coverage reports
10. Verify sample identity	VerifyBamID2, Picard CrosscheckFingerprints	Detect swaps or cross-sample contamination	tumor.bqsr.bam, normal.bqsr.bam, common SNP resource	concordance metrics, contamination estimates
11. Pileup summaries	GATK GetPileupSummaries	Calculate allele fractions at common SNP sites	tumor.bqsr.bam, normal.bqsr.bam, small_exac_common_3.hg38.vcf.gz, hg38 reference	tumor.pileups.table, normal.pileups.table
12. Contamination estimation	GATK CalculateContamination	Estimate tumor contamination and segmentation for Mutect2 filtering	pileup tables	contamination.table, segments.table
13. Mutect2 somatic calling	GATK Mutect2	Detect somatic SNVs/indels using probabilistic caller	tumor.bqsr.bam, normal.bqsr.bam, hg38 reference, af-only-gnomad.hg38.vcf.gz, pon.vcf.gz, exome BED	mutect2.unfiltered.vcf.gz, mutect2.f1r2.tar.gz
14. Orientation bias model	GATK LearnReadOrientationModel	Learn FFPE/orientation artifact signatures	mutect2.f1r2.tar.gz	read-orientation-model.tar.gz
15. Mutect2 filtering	GATK FilterMutectCalls	Filter probable artifacts and germline calls	mutect2.unfiltered.vcf.gz, contamination.table, orientation model, segmentation table	mutect2.filtered.vcf.gz
16. DeepSomatic calling	DeepSomatic	Detect somatic SNVs/indels using deep-learning caller	tumor.bqsr.bam, normal.bqsr.bam, hg38 reference, exome BED	deepsomatic.vcf.gz
17. Variant normalization	bcftools norm, vt	Left-align and normalize variants	Mutect2 and DeepSomatic VCFs, hg38 reference	normalized VCFs
18. Variant merging/consensus	bcftools merge/isec, custom scripts	Build high-confidence consensus somatic callset	normalized Mutect2 VCF, normalized DeepSomatic VCF	somatic.consensus.vcf.gz, somatic.unique_calls.vcf.gz
19. Variant annotation	VEP or Funcotator, bcftools annotate	Add functional, population, and cancer annotations	consensus VCF, VEP cache, COSMIC, ClinVar, dbNSFP	somatic.annotated.vcf.gz, annotation TSV/MAF
20. CNVkit reference creation	CNVkit batch/reference	Build pooled normal reference and coverage model for WES CN calling	normal BAMs, exome target BED, antitarget BED, hg38 reference	cnvkit_reference.cnn
21. CNVkit target coverage	CNVkit coverage	Calculate read depth in target regions	tumor.bqsr.bam, target BED	tumor.targetcoverage.cnn
22. CNVkit antitarget coverage	CNVkit coverage	Calculate off-target/background coverage	tumor.bqsr.bam, antitarget BED	tumor.antitargetcoverage.cnn
23. CNVkit fix	CNVkit fix	Normalize and bias-correct coverage using pooled normal reference	target/antitarget coverage files, cnvkit_reference.cnn	tumor.cnr
24. CNVkit segmentation	CNVkit segment	Segment normalized copy-number signal	tumor.cnr	tumor.cns
25. CNVkit copy-number calling	CNVkit call	Infer integer CN states using purity/ploidy-aware calling	tumor.cnr, tumor.cns, purity estimate, ploidy estimate	called .cns, .seg, CNA tables
26. CNVkit purity/ploidy estimation	CNVkit call, CNVkit BAF utilities	Estimate tumor purity and ploidy from CN profiles and BAFs	segmented CNVkit outputs, optional VCF with heterozygous SNPs	purity/ploidy metrics, scatter plots
27. B-allele frequency extraction	bcftools mpileup, CNVkit	Compute BAFs for allele-specific CN analysis	tumor/normal BAMs, common SNP VCF	BAF tables
28. Tumor-in-normal estimation	deTiN	Estimate tumor contamination in matched normal	somatic VCF, CNVkit segmentation, tumor/normal BAMs	tin_estimate.txt, rescued somatic calls
29. Optional orthogonal CNA analysis	Sequenza	Independent CNA/purity/ploidy validation	tumor.bqsr.bam, normal.bqsr.bam, hg38 reference, GC wiggle files	sequenza_segments.txt, sequenza_purity_ploidy.txt
30. CNA reconciliation	Custom R/Python scripts, manual review	Compare CNVkit and optional Sequenza estimates	CNVkit + Sequenza outputs	final_cna_summary.tsv, reconciled purity/ploidy
31. Final reporting	MultiQC, RMarkdown, Python notebooks, custom reporting scripts	Produce integrated somatic/CNA report	annotated variants, CNVkit outputs, purity/ploidy, contamination, TiN, QC metrics	final PDF/HTML/TSV/JSON report

⸻

2. Post-Variant Filtering and Rescue Workflow

This section describes a practical strategy for:

* filtering artifacts
* removing likely germline variants
* integrating caller concordance
* incorporating CNV/purity information
* rescuing true somatic variants using deTiN

⸻

Key concepts

Cross-sample contamination vs Tumor-in-Normal (TiN)

These are different phenomena.

Type	Meaning
Cross-sample contamination	DNA contamination from another sample/library
Tumor-in-normal (TiN)	Tumor DNA present in the matched normal sample

CalculateContamination estimates cross-sample contamination.

deTiN estimates tumor-derived contamination in the normal.

⸻

Example TiN rescue logic

Suppose:

Metric	Value
Tumor VAF	35%
Normal VAF	4%
Tumor purity	70%
TiN estimate	8%

Expected normal VAF from TiN:

0.35 × 0.08 ≈ 2.8%

Observed normal VAF (4%) is compatible with TiN contamination.

Therefore the variant may be rescued as somatic instead of discarded as germline/artifact.

⸻

Post-Filtering Workflow Table

Step	Stage	Goal	Inputs	Output	Automated tool vs custom/manual
1	Candidate collection	Keep a broad variant universe before hard filtering	mutect2.unfiltered.vcf.gz, mutect2.filtered.vcf.gz, deepsomatic.vcf.gz	candidate_variants.vcf.gz	Custom script
2	Mutect2 artifact filtering	Apply standard Mutect2 filters using contamination and orientation-bias models	Mutect2 VCF, contamination.table, read-orientation-model.tar.gz, segments.table	mutect2.filtered.vcf.gz	Mostly automated via FilterMutectCalls
3	DeepSomatic filter parsing	Separate DeepSomatic PASS, germline-like, and low-confidence calls	deepsomatic.vcf.gz	deepsomatic.pass.vcf.gz, deepsomatic.nonpass.vcf.gz	Mostly automated
4	Caller concordance	Tier variants by caller support	Mutect2 filtered VCF, DeepSomatic VCF	caller_tiers.tsv	Custom script
5	PON/artifact removal	Remove recurrent artifacts	candidate_variants.vcf.gz, pon.vcf.gz	PON-filtered candidate set	Automated + custom
6	Germline AF filtering	Remove likely germline variants	gnomAD AF resource, VCF annotations	germline-filtered VCF/TSV	Automated/custom
7	Technical quality filtering	Remove poor-support calls	tumor/normal depth, base quality, mapping quality, strand bias, duplicate support, local realignment context	QC-filtered candidate table	Mostly custom thresholds
8	Normal-evidence review	Identify variants rejected because the normal has ALT reads	Mutect2 filter tags, tumor/normal VAFs	normal_alt_candidates.tsv	Custom script
9	CNV-aware VAF check	Check whether tumor VAF is compatible with local copy number and purity	CNVkit .cns/.call.cns, tumor purity, ploidy, variant VAFs	CNV-aware classification	Custom/manual review
10	deTiN TiN estimation	Estimate tumor-in-normal contamination	candidate SSNVs, indels, allele counts, CN states	tin_estimate.txt	Automated via deTiN
11	deTiN rescue	Rescue variants whose normal ALT signal is compatible with TiN	normal_alt_candidates.tsv, tin_estimate.txt, CNVkit segments, tumor/normal VAFs	tin_rescued_variants.tsv	deTiN + custom post-processing
12	DeepSomatic-only rescue	Recover plausible variants missed/filtered by Mutect2	DeepSomatic PASS calls, Mutect2 filtered/non-emitted status, PON, gnomAD, normal VAF	deepsomatic_rescue.tsv	Custom/manual review
13	Mutect2-only review	Keep or downgrade Mutect2-only calls	Mutect2 PASS calls absent from DeepSomatic	mutect2_only_review.tsv	Custom/manual review
14	FFPE/artifact blacklist	Remove oxidative, FFPE, low-complexity, homopolymer, or bait-edge artifacts	sequence context, orientation bias tags, target BED, mappability/blacklist BEDs	artifact-filtered set	Partly automated, often custom/manual
15	CNA consistency check	Ensure SNV/indel interpretation agrees with CN state	CNVkit segments, purity/ploidy, VAFs	flagged discordant variants	Custom/manual
16	Final tiering	Assign final reporting tiers	all caller, QC, CNV, TiN, annotation data	final_somatic_variants.tsv, final_somatic.vcf.gz	Custom script
17	Manual IGV review	Inspect borderline or clinically important variants	BAMs, final candidate list, CNV plots	reviewer decisions	Manual
18	Final report	Summarize accepted, rescued, and rejected calls	final VCF/TSV, CNVkit outputs, purity/ploidy, TiN, contamination	final report	Custom/reporting tool

⸻

Recommended Final Variant Categories

Category	Definition
PASS_CONSENSUS	PASS in both Mutect2 and DeepSomatic
PASS_MUTECT2_ONLY	Mutect2 PASS, DeepSomatic absent or filtered
PASS_DEEPSOMATIC_ONLY	DeepSomatic PASS, Mutect2 absent or filtered
RESCUED_TIN	Initially suspicious because of normal ALT reads, but compatible with deTiN estimate
RESCUED_CNV_AWARE	VAF initially appeared inconsistent until interpreted using CN/purity context
LIKELY_GERMLINE	Normal signal and population AF inconsistent with somatic origin
LIKELY_ARTIFACT	PON hit, strand/orientation artifact, low complexity, poor mapping, or bait-edge artifact
REVIEW_REQUIRED	Biologically plausible but technically borderline

⸻

Example TiN Rescue Rule

Rescue as RESCUED_TIN if:
  Mutect2 filter is germline / normal_artifact / alt_allele_in_normal
  AND tumor VAF >= 0.05
  AND normal VAF <= tumor VAF × TiN + tolerance
  AND variant is not common in gnomAD
  AND variant is not in PON
  AND local CN state supports the observed tumor VAF

⸻

Recommended Modern Workflow Architecture

FASTQ
  → alignment
  → duplicate marking
  → BQSR
  → Mutect2 + DeepSomatic
  → variant normalization + consensus
  → CNVkit purity/ploidy/CNA
  → deTiN refinement
  → TiN-aware rescue filtering
  → final somatic VCF + CNA report

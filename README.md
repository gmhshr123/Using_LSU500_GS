
I decided to proceed with the analysis, even though I haven't yet received a response from LSU. The data from Stan (originally from a previous postdoc) has some issues, especially with chromosome numbers and SNP positions. Some positions that were in the original data are missing here.

So, I removed the lines with those missing positions and converted everything to VCF format using BCFtools. I ran into a problem on the UARK HPC: it was missing the libcrypto.so.1.0.0 library, which BCFtools needs. I got around this by loading the samtools module (ml samtools), which provides the necessary BCFtools functionality.

The processed files, SIT_removeNA_reordered.vcf.gz and Sorted_Rice_genotyp_Sha.vcf.gz, are now located in /scrfs/storage/minghaog/LSU500.

Another issue arose: two markers, LSU_diversity_86 and LSU_diversity_77, had the exact same position (23439757) on chromosome 1. As a temporary workaround, I changed the position of LSU_diversity_77 to 23439758. I understand this data isn't ideal for a robust analysis, but it allows me to start getting familiar with the GS workflow.

After that, I compressed the VCF files using bgzip, indexed them with bcftools index, and then merged them into merged_vcf_520.vcf using bcftools merge.

bgzip SIT_removeNA_reordered.vcf
bgzip Sorted_Rice_genotyp_Sha.vcf
bcftools index SIT_removeNA_reordered.vcf.gz
bcftools index Sorted_Rice_genotyp_Sha.vcf.gz
bcftools merge SIT_removeNA_reordered.vcf.gz Sorted_Rice_genotyp_Sha.vcf.gz -o merged_vcf_520.vcf

I used vcftools with --max-missing 0.8 to check for missing data, which resulted in 490 out of 519 sites being retained. I then filtered for minor allele frequency (MAF) using vcftools --maf 0.05, which left me with 439 of those 490 sites.
vcftools --vcf merged_vcf_520.vcf --max-missing 0.8 --recode --out merged_nomiss08
vcftools --vcf merged_nomiss08.recode.vcf --maf 0.05 --recode --out merged_nomiss08maf005

Finally, I got the real original spreadsheet of LSU550 about their ID and CHROM and POS. 
import pandas as pd
import argparse

def convert_genotype(genotype, ref, alt):
    """Convert genotype representation to VCF format (0/0, 0/1, 1/1, ./.)"""
    if pd.isna(genotype):
        return "./."
    elif genotype == ref * 2:
        return "0/0"
    elif genotype == alt * 2:
        return "1/1"
    elif (genotype[0] == ref and genotype[1] == alt) or (genotype[0] == alt and genotype[1] == ref):
        return "0/1"
    else:
        return "./."

def convert_csv_to_vcf(input_csv, output_vcf):
    """Convert a genotype CSV file to VCF format."""
    # Load CSV
    df = pd.read_csv(input_csv)
    
    # Define VCF header
    vcf_header = """##fileformat=VCFv4.2
##source=CustomConversion
##FORMAT=<ID=GT,Number=1,Type=String,Description="Genotype">
#CHROM\tPOS\tID\tREF\tALT\tQUAL\tFILTER\tINFO\tFORMAT\t""" + "\t".join(df.columns[5:])

    # Prepare VCF content
    vcf_rows = []
    for _, row in df.iterrows():
        chrom, pos, snp_id, ref, alt = row[:5]
        genotypes = [convert_genotype(row[sample], ref, alt) for sample in df.columns[5:]]
        vcf_rows.append(f"{chrom}\t{pos}\t{snp_id}\t{ref}\t{alt}\t.\t.\t.\tGT\t" + "\t".join(genotypes))

    # Write VCF file
    with open(output_vcf, "w") as vcf_file:
        vcf_file.write(vcf_header + "\n")
        vcf_file.write("\n".join(vcf_rows))

def main():
    parser = argparse.ArgumentParser(description="Convert a CSV genotype file to VCF format.")
    parser.add_argument("input_csv", help="Path to input CSV file")
    parser.add_argument("output_vcf", help="Path to output VCF file")
    args = parser.parse_args()
    
    convert_csv_to_vcf(args.input_csv, args.output_vcf)

if __name__ == "__main__":
    main()



Still, I change it to the VCF first, which I mostly comfortable with. 

[![Gitpod ready-to-code](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/renatopuga/somatico)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1r4LDUiQqirUFQT2nIrhVW_AHLaBhmS7y?usp=sharing)


# Análise Somática
GATK 4 Mutect2 Somático

# Aula Prática

![image](https://user-images.githubusercontent.com/8321336/130251648-7ad77cae-435f-49be-950f-b7af5f59fd7a.png)

## Pipeline

* GATK4 - Mutect2
* Gene JAK2
* Referência chr9
  * Sobre as versões do Genoma Humano: https://gatk.broadinstitute.org/hc/en-us/articles/360035890711-GRCh37-hg19-b37-humanG1Kv37-Human-Reference-Discrepancies#grch37
* Amostras: 
  * WP043 (tumor)
  * WP044 (normal)
* https://gatk.broadinstitute.org/hc/en-us/articles/360035894731-Somatic-short-variant-discovery-SNVs-Indels-


## Amostras Extras

- WP190 (tumor) e WP191 (normal)
- WP017 (tumor) e WP018 (normal)

**Nota 1:** Utilizar o af-gnomad chr13 e chr19. 

**Nota 2:** Será preciso baixar o chr13 e chr19 da UCSC.

**Download chr19**
```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz
```

**Download chr13**
```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz
```



**Concatenar os arquivos .fa.gz**
> Dica 1: zcat lê arquivos compactados .gz e zip

```bash
zcat chr13.fa.gz chr19.fa.gz | sed -e "s/chr//g" > hg19.fa
```

**Gerar o index do arquivo hg19.fa**

** instalar o samtools** 


```bash
sudo apt-get update
sudo apt-get install -y samtools
samtools faidx hg19.fa
```
** salvar historico de comandos **
history > historico_comandos.txt
 ____________________ 
< Primeiro é o chr9 >
 -------------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||



## Workflow
Os arquivos BAM (tumor e normal) já foram gerados e estão prontos para a chamada de variates (ver Anexo 1). Então, agora vamos fazer download da referência `chr9` e gerar o index com o programa `samtools`.

### Download da Referência - chr9

* Download 

```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr9.fa.gz
```

* Alterar nome do header: DE: >chr9 para >9
> Essa alteração é necessária pois no BAM a referência não tinha `>chr` era apenas `>9`.

```bash
zcat chr9.fa.gz | sed -e "s/chr//g" > chr9.fa
```

* Verificar se alteração foi feita com o comando `head`

```bash
head chr9.fa
```
> O comando `head` lê as 10 primeiras linha de um arquivo texto


## samtools

Samtools is a suite of programs for interacting with high-throughput sequencing data. It consists of three separate repositories:



### samtools install

* samtools install (Mac)

```bash
brew install samtools 
```

* samtools install (Ubuntu)

```bash
sudo apt-get install samtools 
```

* samtools install (Docker)

```bash
docker pull biocontainers/samtools
```



### samtools faidx e index

* samtools faidx

```bash
samtools faidx chr9.fa
```


## GATK4

> Version: 4.2.2.0

Genome Analysis Toolkit - Variant Discovery in High-Throughput Sequencing Data. https://gatk.broadinstitute.org/



### GATK4 install

GATK4 install Docker

```bash
docker pull broadinstitute/gatk:4.2.2.0
```



GATK4 install Mac e Linux

* Download

```bash
wget -c https://github.com/broadinstitute/gatk/releases/download/4.2.2.0/gatk-4.2.2.0.zip
```

* Descompactar

```bash
unzip gatk-4.2.2.0.zip 
```

* Testando gatk

```bash
./gatk-4.2.2.0/gatk
```



### GATK4 .dict

```bash
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R chr9.fa -O chr9.dict
```



### GATK4 intervals

```bash
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R chr9.fa -O chr9.interval_list -OT ACGT
```



## Mutect2

Call somatic SNVs and indels via local assembly of haplotypes



### Mutect2 Tumor e Normal
> O comando: `samtools view -H normal_JAK2.bam` você consegue pegar o campo SM: que contém o ID da amostra normal.

```bash
samtools view -H normal_JAK2.bam | grep RG | cut -f6 | sed -e "s/SM://g"
```


```bash
./gatk-4.2.2.0/gatk Mutect2 \
	-R chr9.fa \
	-I tumor_JAK2.bam \
	-I normal_JAK2.bam \
	-normal WP044 \
	--germline-resource af-only-gnomad-chr9.vcf.gz \
	-O somatic.vcf.gz \
	-L chr9.interval_list
```



## Calcular Contaminação



### GetPileupSummaries

Tabulates pileup metrics for inferring contamination

* GetPileupSummaries Tumor

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I tumor_JAK2.bam \
	-V af-only-gnomad-chr9.vcf.gz \
	-L chr9.interval_list \
	-O tumor_JAK2.table
```

* GetPileupSummaries Normal

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I normal_JAK2.bam \
	-V af-only-gnomad-chr9.vcf.gz \
	-L chr9.interval_list \
	-O normal_JAK2.table
```



### CalculateContamination

Calculate the fraction of reads coming from cross-sample contamination

```bash
./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_JAK2.table \
	-matched normal_JAK2.table \
	-O contamination.table
```



### FilterMutectCalls

Filter somatic SNVs and indels called by Mutect2

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R chr9.fa \
	-V somatic.vcf.gz \
	--contamination-table contamination.table \
	-O filtered.vcf.gz
```
## verificar informações sobre cromosso, posição genômica, alelo

zgrep -v '^#' filtered.vcf.gz | cut -f1,2,4,5,7

## VEP ensembl - Anotação

### VEP install

```bash
docker pull ensemblorg/ensembl-vep
```

Criar o diretorio vep_output
```
mkdir -p vep_output
```

Modificar a permissao do diretorio vep_output
```
chmod 777 vep_output
```

* Aplicar apenas no Google Colab

```bash
mkdir -p vep_output
chmod 777 vep_output
```

# rodar o vep

```bash
bash vep-docker.sh
```
# confirmar se o arquivo foi criado

ls -lh vep_output/

# verificar as anotações
cat vep_output/filtered.vep.tsv

## Panel of Normal (PoN)


# Panel of Normal (PoN)

GATK Best Practices - Exome PoN


* vcf

```bash
wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf
```

* vcf.idx

```bash
wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf.idx
```
## criar dicionário
./gatk-4.2.2.0/gatk CreateSequenceDictionary \
    -R hg19.fa \
    -O hg19.dict

## Gerar os intervalos genômicos

./gatk-4.2.2.0/gatk ScatterIntervalsByNs \
    -R hg19.fa \
    -O hg19.interval_list \
    -OT ACGT


# verificar a referencia
cut -f1,2 hg19.fa.fai

# Verificar o BAM da amostra WP190:
samtools view -H tumor_wp190.bam | grep '^@SQ'

# Verificar o painel de normais:
grep '^##contig=' Mutect2-exome-panel.vcf | head



* Mutect2

```bash
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19.interval_list \
  -O WP190.somatic.pon.vcf.gz
  
```

* CalculateContamination somente com o table do tumor (ex.: wp190)

```bash
./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp190.table \
	-O WP190.contamination.pon.table
```

* FilterMutectCalls

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V WP190.somatic.pon.vcf.gz \
	--contamination-table WP190.contamination.pon.table \
	-O WP190.filtered.pon.vcf.gz
```

## corrigindo o erro da possível pegadinha
./gatk-4.2.2.0/gatk GetPileupSummaries \
    -I tumor_wp190.bam \
    -V af-only-gnomad-chr13-chr19.vcf.gz \
    -L hg19.interval_list \
    -O tumor_wp190.table

head tumor_wp190.table

* CalculateContamination somente com o table do tumor (ex.: wp190)

```bash
./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp190.table \
	-O WP190.contamination.pon.table
```

* FilterMutectCalls

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V WP190.somatic.pon.vcf.gz \
	--contamination-table WP190.contamination.pon.table \
	-O WP190.filtered.pon.vcf.gz
```

## verificar as variantes filtradas
zgrep -v '^#' WP190.filtered.pon.vcf.gz | cut -f1,2,4,5,7


zgrep -v '^#' WP190.filtered.pon.vcf.gz | awk -F'\t' '$7=="PASS"' | wc -l

samtools idxstats tumor_wp190.bam | grep -E '^(13|19)[[:space:]]'



./gatk-4.2.2.0/gatk Mutect2 \
    -R hg19.fa \
    -I tumor_wp017.bam \
    -I tumor_wp018.bam \
    -normal WP018 \
    --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
    -L hg19.interval_list \
    -O WP017.somatic.vcf.gz

# Anexo 1

### Converter BAM para FASTQ

```bash
samtools view -h -b /Volumes/Seagate\ Expansion\ Drive/data-lpfap10/projects/proadi/exome/bam/WP043/WP043.bam 9:5021937-5126899 | samtools bam2fq -1 tumor_R1.fq -2 tumor_R2.fq - 
```

```bash
samtools view -h -b /Volumes/Seagate\ Expansion\ Drive/data-lpfap10/projects/proadi/exome/bam/WP044/WP044.bam 9:5021937-5126899 | samtools bam2fq -1 normal_R1.fq -2 normal_R2.fq - 
```



### Converter BAM para JAK2 BAM

Aqui estão os comandos que foram utilizados para gerar os BAMs intermediários e como gerar arquivos FASTQs de regiões específicas do seu arquivo BAM completo.
> Essas etapas não precisam ser executadas nesse pipeline

* BAM para BAM

```bash
samtools view -h -b /Volumes/Seagate\ Expansion\ Drive/data-lpfap10/projects/proadi/exome/bam/WP043/WP043.bam 9:5021937-5126899 > tumor_JAK2.bam
```

```bash
samtools view -h -b /Volumes/Seagate\ Expansion\ Drive/data-lpfap10/projects/proadi/exome/bam/WP044/WP044.bam 9:5021937-5126899 > normal_JAK2.bam
```

* Gerar index do BAM (.BAI)

```bash
samtools index tumor_JAK2.bam 
```

```bash
samtools index normal_JAK2.bam 
```



### af-only-gnomad.vcf.gz (apenas região JAK2)
> [Google Clou af-only-gnomad.vcf.gz](https://console.cloud.google.com/storage/browser/gatk-best-practices/somatic-b37;tab=objects?project=broad-dsde-outreach&pageState=(%22StorageObjectListTable%22:(%22f%22:%22%255B%255D%22))&forceOnObjectsSortingFiltering=false&pli=1)

* Header do VCF

```bash
zgrep -w "\#" af-only-gnomad.raw.sites.chr.vcf.gz > header
```

* Apenas Região do Gene JAK2

```bash
zgrep -w "^chr9" af-only-gnomad.raw.sites.chr.vcf.gz  | awk '$2>=5021937 && $2<=5126899' > JAK2.region
```

* Concatenar header + vcf

```bash
cat header JAK2.region > af-only-gnomad-chr9.vcf
```

* Compactar

```bash
bgzip af-only-gnomad-chr9.vcf
```

* Index do VCF

```bash
tabix -p vcf af-only-gnomad-chr9.vcf.gz 
```

# Anexo 2 – Execução manual do pipeline de identificação e filtragem de variantes somáticas utilizando GATK Mutect2
# As análises foram realizadas por meio da execução individual dos comandos do pipeline fornecido pelo professor, contemplando duas abordagens: análise pareada tumor-normal e análise utilizando Panel of Normals (PoN).

# antes de iniciar, vou conferir os arquivos existentes
ls -lh run.chr13-chr19.sh run.chr13-chr19.pon.sh

cat run.chr13-chr19.sh

cat run.chr13-chr19.pon.sh

grep '^>' hg19.fa

# quantos intervalos foram criados
awk '!/^@/ {print $1}' hg19.interval_list | sort -n | uniq -c

# confirmar os identificadores das amostras no BAM

samtools view -H tumor_wp190.bam | grep '^@RG'
samtools view -H tumor_wp191.bam | grep '^@RG'


# rodar o Mutect2 para o par WP190/WP191
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  -I tumor_wp191.bam \
  -normal WP191 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.somatic.vcf.gz

# conferir
zgrep -vc '^#' WP190.somatic.vcf.gz

# gerar a tabela do tumor WP190

./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp190.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.table

# gerar a tabela da amostra normal WP191
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp191.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP191.table
# conferir tabelas geradas
head -n 5 WP190.table WP191.table

# calculateContamination, estimar a contaminação no tumor WP190
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP190.table \
  -matched WP191.table \
  -O WP190.contamination.table

# conferindo
cat WP190.contamination.table

# filtrar as chamadas do par WP190/WP191
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP190.somatic.vcf.gz \
  --contamination-table WP190.contamination.table \
  -O WP190.filtered.vcf.gz

# ver quantas variantes passaram no filtro
zcat WP190.filtered.vcf.gz | awk '!/^#/ {print $7}' | sort | uniq -c

# confirmar se é 5 mesmo
zgrep -vc '^#' WP190.filtered.vcf.gz

# análise pareada WP017/WP018
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  -I tumor_wp018.bam \
  -normal WP018 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.somatic.vcf.gz

# contar as variantes candidatas
  zgrep -vc '^#' WP017.somatic.vcf.gz

# gerar as tabelas de contagem:
# tumor WP017:
 ./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp017.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.table

# normal WP018:
  ./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp018.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP018.table

# conferir os arquivos
  head -n 5 WP017.table WP018.table

# calcular contaminação
  ./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP017.table \
  -matched WP018.table \
  -O WP017.contamination.table

# conferir resultado
  cat WP017.contamination.table

# filtrar as variantes identificadas pelo Mutect2
  ./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP017.somatic.vcf.gz \
  --contamination-table WP017.contamination.table \
  -O WP017.filtered.vcf.gz

# conferir após o filtro
  zcat WP017.filtered.vcf.gz | awk '!/^#/ {print $7}' | sort | uniq -c

# identificar as variantes PASS

  for vcf in WP190.filtered.vcf.gz WP017.filtered.vcf.gz; do
  printf '\n%s\n' "$vcf"
  zcat "$vcf" | awk '/^#CHROM/ || (!/^#/ && $7 == "PASS")'
done

# executar WP017 com PoN
sh run.chr13-chr19.pon.sh tumor_wp017.bam tumor_wp018.bam WP017 WP018

# conferir as variantes do PASS
zgrep -v '^#' WP017.filtered.pon.vcf.gz | \
awk -F'\t' '{print $7}' | sort | uniq -c

# contar somente as variantes PASS do WP017
zcat WP017.filtered.vcf.gz | \
awk -F'\t' '!/^#/ && $7=="PASS"' | wc -l

# conferir estimativa de contaminação
cat WP017.contamination.pon.table

# confirmar a referência dos BAMs:
samtools idxstats tumor_JAK2.bam | grep -E '^(9|13|19)[[:space:]]'
samtools idxstats normal_JAK2.bam | grep -E '^(9|13|19)[[:space:]]'

# executar Mutect2 para WP043 com PoN
## Conferir os arquivos disponíveis
ls -lh tumor_JAK2.bam normal_JAK2.bam

# conferir os identificadores das amostras
samtools view -H tumor_JAK2.bam | grep '^@RG'
samtools view -H normal_JAK2.bam | grep '^@RG'

# conferir a referência atual
grep '^>' hg19.fa

# preparar a referência do cromossomo 9
## baixar o cromossomo 9 (hg19)
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr9.fa.gz

# criar uma referência específica para JAK2
zcat chr9.fa.gz | sed 's/^>chr/>/' > hg19_chr9.fa

# indexar a referência
samtools faidx hg19_chr9.fa

# criar o dicionário
./gatk-4.2.2.0/gatk CreateSequenceDictionary \
  -R hg19_chr9.fa \
  -O hg19_chr9.dict

# conferir os arquivos gerados
ls -lh hg19_chr9.fa hg19_chr9.fa.fai hg19_chr9.dict

grep '^>' hg19_chr9.fa

cat hg19_chr9.fa.fai

# gerar a lista de intervalos do cromossomo 9
./gatk-4.2.2.0/gatk ScatterIntervalsByNs \
  -R hg19_chr9.fa \
  -O hg19_chr9.interval_list \
  -OT ACGT

# verificar se o arquivo foi criado
ls -lh hg19_chr9.interval_list

# conferir os intervalos gerados
awk '!/^@/ {print $1}' hg19_chr9.interval_list | sort | uniq -c


# verificar os arquivos gnomAD disponíveis
ls -lh *gnomad*

# conferir os cromossomos presentes no gnomAD
zgrep -v '^#' af-only-gnomad-chr9.vcf.gz | \
cut -f1 | sort -u

# contar as variantes disponíveis
zgrep -vc '^#' af-only-gnomad-chr9.vcf.gz

# conferir o cabeçalho do VCF
zgrep '^##contig=' af-only-gnomad-chr9.vcf.gz | head

# executar Mutect2 para WP043/WP044 (JAK2)
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19_chr9.fa \
  -I tumor_JAK2.bam \
  -I normal_JAK2.bam \
  -normal WP044 \
  --germline-resource af-only-gnomad-chr9.vcf.gz \
  -L hg19_chr9.interval_list \
  -O WP043.somatic.vcf.gz

# contar as variantes candidatas identificadas
  zgrep -vc '^#' WP043.somatic.vcf.gz

# gerar as tabelas de contagem do tumor e do normal
# gerar a tabela do tumor WP043
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_JAK2.bam \
  -V af-only-gnomad-chr9.vcf.gz \
  -L hg19_chr9.interval_list \
  -O WP043.table

# gerar a tabela da amostra normal WP044
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I normal_JAK2.bam \
  -V af-only-gnomad-chr9.vcf.gz \
  -L hg19_chr9.interval_list \
  -O WP044.table

# conferir as tabelas geradas
  head -n 5 WP043.table WP044.table

# estimar a contaminação no tumor WP043
  ./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP043.table \
  -matched WP044.table \
  -O WP043.contamination.table

# conferir a estimativa da contaminação
  cat WP043.contamination.table

# filtrar as variantes do par WP043/WP044

./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19_chr9.fa \
  -V WP043.somatic.vcf.gz \
  --contamination-table WP043.contamination.table \
  -O WP043.filtered.vcf.gz

# Contar as variantes por categoria de filtragem
zcat WP043.filtered.vcf.gz | \
awk '!/^#/ {print $7}' | sort | uniq -c

# contar o total de variantes no VCF filtrado
zgrep -vc '^#' WP043.filtered.vcf.gz

# contar somente as variantes PASS
zcat WP043.filtered.vcf.gz | \
awk -F'\t' '!/^#/ && $7=="PASS"' | wc -l

# executar Mutect2 para WP043 com PoN
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19_chr9.fa \
  -I tumor_JAK2.bam \
  --germline-resource af-only-gnomad-chr9.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19_chr9.interval_list \
  -O WP043.somatic.pon.vcf.gz

# contar as variantes candidatas identificadas com PoN
zgrep -vc '^#' WP043.somatic.pon.vcf.gz

# gerar a tabela do tumor WP043
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_JAK2.bam \
  -V af-only-gnomad-chr9.vcf.gz \
  -L hg19_chr9.interval_list \
  -O WP043.pon.table

# Estimar a contaminação
  ./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP043.pon.table \
  -O WP043.contamination.pon.table

# conferir
  cat WP043.contamination.pon.table

# Filtrar as variantes com PoN
  ./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19_chr9.fa \
  -V WP043.somatic.pon.vcf.gz \
  --contamination-table WP043.contamination.pon.table \
  -O WP043.filtered.pon.vcf.gz

# Contar as variantes por categoria de filtragem
zcat WP043.filtered.pon.vcf.gz | \
awk '!/^#/ {print $7}' | sort | uniq -c

# Contar o total de variantes
zgrep -vc '^#' WP043.filtered.pon.vcf.gz

# Contar somente as variantes PASS
zcat WP043.filtered.pon.vcf.gz | \
awk -F'\t' '!/^#/ && $7=="PASS"' | wc -l

# identificar as 14 variantes PASS da WP043 com PoN, incluindo suas posições genômicas.
zcat WP043.filtered.pon.vcf.gz | \
awk -F'\t' '!/^#/ && $7=="PASS" {print $1, $2, $4, $5, $7}'

# FIM

## INTERPRETAÇÃO DOS RESULTADOS
# WP043: apresentou a maior diferença entre as abordagens, com 1 variante PASS na análise pareada e 14 na análise com PoN.
# WP017: apresentou 1 variante PASS em cada abordagem.
# WP190: apresentou 2 variantes PASS na análise pareada e 3 com PoN

# ----------------------------------------

# ANEXO - script disponibilizado pelo professor

# Rodar amostras extras pareadas:

* Com par:
```bash
sh run.chr13-chr19.sh tumor_JAK2.bam normal_JAK2.bam WP043 WP044
sh run.chr13-chr19.sh tumor_wp017.bam tumor_wp018.bam WP017 WP018
sh run.chr13-chr19.sh tumor_wp190.bam tumor_wp191.bam WP190 WP191
```

* Com PoN:
```bash
sh run.chr13-chr19.pon.sh tumor_JAK2.bam normal_JAK2.bam WP043 WP044 
sh run.chr13-chr19.pon.sh tumor_wp017.bam tumor_wp018.bam WP017 WP018 
sh run.chr13-chr19.pon.sh tumor_wp190.bam tumor_wp191.bam WP190 WP191 

```

**Código Fonte (run.chr13-chr19.pon.sh e run.chr13-chr19.sh)**

```bash

# variaveis fixas
gnomad="af-only-gnomad-chr13-chr19.vcf.gz"
interval="hg19.interval_list"
genome="hg19.fa"

# bam e ids 
tumor=$1
normal=$2
id_tumor=$3
id_normal=$4

# rodando mutect2
./gatk-4.2.2.0/gatk Mutect2 \
	-R $genome \
	-I $tumor \
	-I $normal \
	-normal $id_normal \
	--germline-resource $gnomad \
	-O $id_tumor.somatic.vcf.gz \
	-L $interval

# getPileup tumor
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I $tumor \
	-V $gnomad \
	-L $interval \
	-O $id_tumor.table

# getPileup normal
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I $normal \
	-V $gnomad \
	-L $interval \
	-O $id_normal.table

# CalculateContamination
./gatk-4.2.2.0/gatk CalculateContamination \
	-I $id_tumor.table \
	-matched $id_normal.table \
	-O $id_tumor.contamination.table

# FilterMutectCalls
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R $genome \
	-V $id_tumor.somatic.vcf.gz \
	--contamination-table $id_tumor.contamination.table \
	-O $id_tumor.filtered.vcf.gz

```

Rodar amostras extras com Panel of Normal (broad institute).

```bash

# variaveis fixas
gnomad="af-only-gnomad-chr13-chr19.vcf.gz"
interval="hg19.interval_list"
genome="hg19.fa"
pon="Mutect2-exome-panel.vcf"

# bam e ids 
tumor=$1
id_tumor=$3

# rodando mutect2
./gatk-4.2.2.0/gatk Mutect2 \
	-R $genome \
	-I $tumor \
	--germline-resource $gnomad \
	--panel-of-normals $pon \
	-O $id_tumor.somatic.pon.vcf.gz \
	-L $interval

# getPileup tumor
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I $tumor \
	-V $gnomad \
	-L $interval \
	-O $id_tumor.pon.table

# CalculateContamination
./gatk-4.2.2.0/gatk CalculateContamination \
	-I $id_tumor.pon.table \
	-O $id_tumor.contamination.pon.table

# FilterMutectCalls
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R $genome \
	-V $id_tumor.somatic.pon.vcf.gz \
	--contamination-table $id_tumor.contamination.pon.table \
	-O $id_tumor.filtered.pon.vcf.gz
```

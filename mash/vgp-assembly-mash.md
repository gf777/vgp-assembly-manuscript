# VGP assembly mash graph
## Download VGP metadata from NCBI
NCBI's dataset utility can be downloaded from [here](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/download-and-install/).
Download VGP Bioproject metadata:
```
datasets summary genome accession PRJNA489243 > vgp-metadata.json
```
## list VGP repo
We can use [jq](https://jqlang.github.io/jq/) to parse the repo (here is jq's [manual](https://jqlang.github.io/jq/manual/)):
```
cat vgp-metadata.json | jq -r '.reports[] | .accession + "," + .assembly_info.assembly_name + "," + (.assembly_info.biosample.attributes[] | select(.name=="scientific_name").value) // .accession + "," + .assembly_info.assembly_name + ","' > accession_metadata.ls
cat accession_metadata.ls
```
## Download assemblies
We can download a random subset of genomes combining jq's and NCBI's datasets functionalities, then compute [mash](https://github.com/marbl/Mash) sketches and all-vs-all distances.
First compute individual mash sketches:
```
#!/bin/bash
set -e

SEED=42
mkdir -p sketches
rm -f genome_list.tsv mash_distances.tsv
while IFS="," read -r accession tolid latin_name
do
	RANDOM=$SEED
	VAL=$RANDOM
	SEED=$RANDOM
	if (( $(echo "scale=4; ${VAL}/32767 > 0.25" |bc -l) )); then
		printf "skipping: $accession\t$tolid\t$latin_name\n"
        continue
    fi
	datasets download genome accession $accession --filename $accession.zip
	unzip -o $accession.zip -d $accession
	printf "$accession\t$tolid\t$latin_name\n" >> genome_list.tsv
	
	genome=$accession/ncbi_dataset/data/$accession/*.fna
	if ! cmp -s <(md5sum $genome | cut -f1 -d' ') <(grep fna $accession/md5sum.txt | cut -f1 -d' '); then
		printf "Check file integrity: $accession"
		exit 1
	fi
	
	mash sketch -s 10000000 $genome
	mv $genome.msh sketches
	rm -r $accession.zip $accession
done<accession_metadata.ls
```
Next compute triangular mash distance matrix:
```
#!/bin/bash

readarray -t in < genome_list.tsv # input list

for (( i=0; i<${#in[@]}; i++ )); do # triangular matrix
    for (( j=0; j<${#in[@]}; j++ )); do
        if [ $j -gt $i ]; then # sketch and get the dist
        	accession1=${in[$i]%%$'\t'*}
        	accession2=${in[$j]%%$'\t'*}
            echo $accession1 $accession2 $(mash dist sketches/$accession1* sketches/$accession2* | cut -f3-5)
        fi
    done
done
```

We can now run `distance_hist.py` to get the histogram of the distances.
We can add metadata to the accessions using this command:
```
awk 'FNR==NR{a[$1]=$2; next} {FS=" "} {$0=$0; print $1, a[$1], a[$2], $2, $3}' FS="," accession_metadata.ls distance_matrix.txt
```
Potentially filtering distant interactions:
```
printf "Accession 1,Tolid 1,Class 1,Accession 2,Tolid 2,Class 2,D\n" > filtered_distance_matrix.txt
awk 'FNR==NR{a[$1]=$2; next} {FS=" "} {$0=$0; if ($3<0.2) printf $1","a[$1]","substr(a[$1], 1, 1)","$2","a[$2]","substr(a[$2], 1, 1)","$3"\n"}' FS="," accession_metadata.ls distance_matrix.txt >> filtered_distance_matrix.txt
```
This can then be visualized in tools such as Cytoscape.
# Import VCF data into BigQuery using Variant Transforms tool
* Authors: Paul Billing-Ross
* Predicted runtime: 1 hour (10,000 VCFs)
* Predicted cost: TBD

## Query the Neo4j graph database to find CNVnator VCFs of participants in the Million Veteran Program (MVP) Whole Genome Sequencing Data Release 1.

Run the following queries in the [Trellis](https://trellis-data-management.readthedocs.io/en/latest/) Neo4j browser to get the Google Cloud Storage URIs for the VCFs that will be included in the analysis. 
Use the browser to download the lists of URIs as a CSV.

### Neo4j Cypher query:
```
MATCH (s:Study {name:"WgsDataRelease1"})-[:HAS_PARTICIPANT]->(:Participant)-[:IS]->(:Person)-[:HAS_BIOLOGICAL_OME]->(:Genome)-[:HAS_SEQUENCING_READS]->(:Cram)-[:WAS_USED_BY]->(:Cnvnator:Job)-[:GENERATED]->(v:Vcf)
RETURN v.uri
```

More information on MVP Whole Genome Sequencing Data Release 1: https://med.stanford.edu/gbsc/vapahcs.html

## Create a new bucket to store the VCFs
Because the VCFs generated by CNVnator are small, and the Variant Transforms tool (https://github.com/googlegenomics/gcp-variant-transforms) only provides the option to use a pattern to find input objects, 
we are going to copy these objects to a new bucket (https://cloud.google.com/storage/docs/gsutil/commands/mb).

```
MY_PROJECT_ID="put-your-project-id-here"
gsutil mb -c standard -l us-west1 -p ${MY_PROJECT_ID} gs://${MY_PROJECT_ID}-wgs-cnvnator
```

## Copy the VCFs listed in a CSV to the new bucket

Example command for CNVnator VCFs to the new bucket:

```
cat {DATESTAMP}-wgs-dr1-cnv-vcf-uris.csv | gsutil -m cp -I gs://${MY_PROJECT_ID}-wgs-cnvnator/data-release-1/
```

## Create a BigQuery dataset to store the variant data

We are going to use the Variant Transforms tool to import and merge the data from the individual VCFs into a set of BigQuery tables, separated by chromosome. 
Before doing this we need to create our BigQuery dataset.

```
bq --location=US --project_id=${MY_PROJECT_ID} mk -d --description "CVNnator variant data generated for participants in the MVP WGS Data Release 1" mvp_wgs_dr1_cnvnator
```

## Create a service account with the necessary permissions to run Variant Transforms

Follow these steps to create a service account called **variant-transforms-invoker**: https://cloud.google.com/life-sciences/docs/how-tos/getting-started.
At step 4 of "Complete the following steps to create a service account key file:" also add the following roles:

* Compute Instance Admin (V1)
  * This will be used to create a virtual machine from which to launch the Variant Transforms job 
* Service Account User
  * Variant Transforms needs access to this service account to create another virtual machine

## Use Variant Transforms tool to import VCF data into BigQuery

### Launch a virtual machine from which to call Variant Transforms

```
gcloud beta compute --project=${MY_PROJECT_ID} instances create cnvnator-vcf-to-bq-invoker --zone=us-west1-b --machine-type=n1-standard-1  --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=variant-transforms-invoker@${MY_PROJECT_ID}.iam.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --image=ubuntu-1604-xenial-v20210429 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-balanced --boot-disk-device-name=cnvnator-vcf-to-bq-invoker --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

### SSH into the virtual machine

Note: If your SSH connection fails with return code 255, check to make sure your firewall setting to allow tcp connections on port 22.

```
gcloud beta compute ssh --zone "us-west1-b" "cnvnator-vcf-to-bq-invoker"  --project "${MY_PROJECT_ID}"
```

### Install Screen and Docker

We are going to run the VCF to Bigquery pipeline using the variant transforms Docker image, so we'll need Docker. I'm also going to install Screen so that I can close the SSH connection from my local computer to the VM without closing the process.

Reference: https://stackoverflow.com/questions/39645118/docker-unable-to-locate-package-docker-engine

```
sudo apt update
# Install Docker
sudo apt install docker.io
# Check that Docker works
sudo docker run hello-world
# Install Screen
sudo apt install screen
```

Launch a new Screen session
```
screen -S variant_transforms
```

### Launch the Variant Transforms job

Reference: https://github.com/googlegenomics/gcp-variant-transforms.

Copy the following script.

```
#!/bin/bash
# Parameters to replace:

GOOGLE_CLOUD_PROJECT=PROJECT_ID
GOOGLE_CLOUD_REGION=us-west1
TEMP_LOCATION=gs://BUCKET_NAME/FILE_PATH/log
INPUT_PATTERN=gs://BUCKET_NAME/FILE_PATH/*.vcf
OUTPUT_TABLE=PROJECT_ID:DATASET_NAME.TABLE_PREFIX

COMMAND="vcf_to_bq \
  --input_pattern ${INPUT_PATTERN} \
  --output_table ${OUTPUT_TABLE} \
  --job_name vcf-to-bigquery \
  --runner DataflowRunner \
  --sample_name_encoding WITH_FILE_PATH \
  --sharding_config_path gcp_variant_transforms/data/sharding_configs/homo_sapiens_fixed_partitions.yaml"

docker run -v ~/.config:/root/.config \
  gcr.io/cloud-lifesciences/gcp-variant-transforms \
  --project "${GOOGLE_CLOUD_PROJECT}" \
  --region "${GOOGLE_CLOUD_REGION}" \
  --temp_location "${TEMP_LOCATION}" \
  "${COMMAND}"
```

Paste the script into a file.

```
# Use the vi text editor to create a new bash script
vi vcf-to-bq.sh
# Copy the (above) script from the Variant Transforms repo and replace it with your environment variables.
```

Launch the variant transforms job.

```
# Change file permission to allow execution
chmod +x vcf-to-bq.sh
# Run job script
sudo ./vcf-to-bq.sh
```

**(Optional)** If you need to close your SSH connection you can detach and reattach to the Screen session when you reconnect.

* Detach from Screen: Hold `Ctrl+a` and press `d`
* Reattach to Screen: Type `screen -r`

Reference: https://linuxize.com/post/how-to-use-linux-screen/


## Exit the VM and delete it

Once your job is finished you can exit the VM and delete it so that it does not continue accruing charges.

```
# From the VM console
exit
```

```
# From your local console
gcloud beta compute --project=${MY_PROJECT_ID} instances delete cnvnator-vcf-to-bq-invoker --zone=us-west1-b
```

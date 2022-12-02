[![CI](https://github.com/bihealth/clinvar-this/actions/workflows/main.yml/badge.svg?branch=main)](https://github.com/bihealth/clinvar-this/actions/workflows/main.yml)
[![codecov](https://codecov.io/gh/bihealth/clinvar-this/branch/main/graph/badge.svg?token=059T45KAQM)](https://codecov.io/gh/bihealth/clinvar-this)
[![Documentation Status](https://readthedocs.org/projects/clinvar-this/badge/?version=latest)](https://clinvar-this.readthedocs.io/en/latest/?badge=latest)
[![Bioconda](https://img.shields.io/conda/dn/bioconda/clinvar-this.svg?label=Bioconda)](https://bioconda.github.io/recipes/clinvar-this/README.html)
[![Pypi](https://img.shields.io/pypi/pyversions/clinvar-this.svg)](https://pypi.org/project/clinvar-this)

# ClinVar This!

ClinVar Submission via API Made Easy

- Free software: MIT license
- Documentation: https://clinvar-this.readthedocs.io/en/latest/

## Getting Started

You will need some experience with VCF files, ClinVar, and the Linux/Mac command line.

### Obtain ClinVar API Key

First of all, you need to register your organisation with NCBI, request a service account, and obtain an API key.
Skip any step if you have already completed it.

1. [Register your organisation with NCBI as they document](https://www.ncbi.nlm.nih.gov/clinvar/docs/api_http/)
2. Send an email to clinvar@ncbi.nlm.nih.gov to request a service account for your organisation.
3. Once you have a service account, create an API key as outlined [at the top of the NCBI ClinVar API documentation](https://www.ncbi.nlm.nih.gov/clinvar/docs/api_http/).

### Install clinvar-this

You can either install the [PyPi package clinvar-this](https://pypi.org/project/clinvar-this/):

```
# pip install clinvar-this
```

Or you install via [conda/bioconda](http://bioconda.github.io/).

```
# conda install -c clinvar-this
```

Check that your installation worked:

```
# clinvar-this --help
Usage: clinvar-this [OPTIONS] COMMAND [ARGS]...

  Main entry point for CLI via click.

Options:
  --verbose / --no-verbose
  --profile TEXT            The profile to use
  --help                    Show this message and exit.

Commands:
  batch   Sub comment category ``batch ...``
  config  Sub command category ``varfish-this config ...``
```

### Configure your API Token

```
# clinvar-this config set auth_token YOUR_AUTH_TOKEN_HERE
```

Check that this worked:

```
# clinvar-this config dump
# path: /home/holtgrem_c/.config/clinvar-this/config.toml
[default]
auth_token = "YOUR_AUTH_TOKEN_HERE"
```

### Prepare a clinvar-this TSV file.

You will need the following header in the first line

- `ASSEMBLY` - the assembly used, e.g., `GRCh37`, `hg19`, `GRCh38`, `hg38`
- `CHROM` - the chromosomal position without `chr` prefix, e.g., `1`
- `POS` - the 1-based position of the first base in `REF` column
- `REF` - the reference allele of your variant
- `ALT` - the alternative allele of your variant
- `OMIM` - the OMIM id of the carrier's condition (not the OMIM gene ID), e.g., `619325`.
  Leave empty or use `not provided` if you have no OMIM ID.
- `MOI` - mode of inheritance, e.g., `Autosomal dominant inheritance` or `Autosomal recessive inheritance`
- `CLIN_SIG` - clinical significance, e.g. `Pathogenic`, or `Likely benign`
- `CLIN_EVAL` - optional, date of late clinical evaluation, e.g. `2022-12-02`, leave empty to fill with the date of today
- `CLIN_COMMENT` - optional, a comment on the clinical significance, e.g., `ACMG Class IV; PS3, PM2_sup, PP4`
- `KEY` - optional, a local key to identify the variant/condition pair.
  Filled automatically with a UUID if missing, recommeded to leave empty.
- `HPO` - List of HPO terms separated by comma or semicolon, any space will be stripped.
  E.g., `HP:0004322; HP:0001263`.

The following shows an example.

```
ASSEMBLY	CHROM	POS	REF	ALT	OMIM	MOI	CLIN_SIG	HPO
GRCh37	19	48183936	C	CA	619325	Autosomal	dominant	inheritance	Likely	pathogenic	HP:0004322;HP:0001263
```

Note that you must use TAB characters (`\t`) for separating the file.

### Import the TSV file into clinvar-this

Use the `batch import` command to import the TSV file into the local clinvar-this storage.

```
# clinvar-this batch import --name=BATCHNAME DATA_FILE.tsv
```

If you do not specify the `--name` parameter then clinvar-this will generate one based on the current time.
This will create a new batch storage folder below `~/.local/share/clinvar-this/default` with the batch name and place a file `payload.$timestamp.json` there.
This corresponds to the data that will be uploaded into ClinVar.

You can now import another TSV file or change your TSV file and re-import it to apply the changes.

### Submit via ClinVar API

Use `batch submit BATCHNAME` to submit the data to the ClinVar API.

```
# clinvar-this batch submit BATCHNAME
```

This will create a new file `submission-response.$timestamp.json` in the batch storage folder.
This file stores the identifier of the ClinVar submission.
This information is subsequently used in `batch retrieve`.

### Retrieve ClinVar API Submission Result

You can now use the following command to query the ClinVar API for the status of your submission.

```
# clinvar batch retrieve BATCHNAME
```

It will get the submission ID from the latest `submission-response.*.json` file (using lexicographic file name comparison) and query the ClinVar API.
The API response will be written to `retrieve-response.$timestamp.json`.
In the case that the API has processed your submission, clinvar-this will create a new `payload.$timestamp.json` file to reflect the change.
You will probably have to wait a few or many minutes until the processing finishes.
This will store any error message or ClinVar SCV.

### Obtain SCV or Error Message

You could now look at the `payload.$timestamp.json` file to see the full server response.
It is more convenient, however, to export the results to a TSV file again which will display the SCV identifiers and any error message:

```
# clinvar-this batch export BATCHNAME DATA_FILE.reply.tsv
```

The [ClinVar API documentation](https://www.ncbi.nlm.nih.gov/clinvar/docs/api_http/) says that variants submitted via the API do not have to pass manual curation.
That is, the server will perform a number of checks.
If your variants pass all checks then you will directly obtain an SCV and the variants will become publically available on the next Sunday.

### Rinse and Repeat

In the case of a partial success, update the exported TSV file and submit it again until you are happy.

## Caveats

- **The `--use-testing` and `--dry-run` mode.**
  When enabling `--use-testing`, an alternative API endpoint provided by ClinVar will be used.
  This endpoint may use a different schema than the official endpoint (e.g., this has happened in November 2022).
  ClinVar has previously notified their submitters via email without official news posts.

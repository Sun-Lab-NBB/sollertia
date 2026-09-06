# Remote job script

The shell script that `sollertia-forgery` composes for each SLURM allocation, and the resource requests it carries.

---

`server/job.py` composes one shell script per allocation and `Server.submit_job` uploads it, marks it executable, and
submits it. The batch directory is `<server data root>/processing_batches/<batch_id>`, holding each allocation's `.sh`
script alongside its `.out` and `.err` logs. A submission naming several prepared batches writes all of them into the
first identifier's directory. The script carries exactly six unconditional SBATCH directives, in this order.

```text
#!/bin/bash
#SBATCH --cpus-per-task=<cores>
#SBATCH --job-name=<NNNN>-<unit>-<job name>-<specifier>
#SBATCH --output=<batch directory>/<NNNN>-<unit>-<job name>-<specifier>.out
#SBATCH --error=<batch directory>/<NNNN>-<unit>-<job name>-<specifier>.err
#SBATCH --mem=<gigabytes>G
#SBATCH --time=<[D-]HH:MM:SS>
[#SBATCH --dependency=afterok:<id>[:<id>…]]
[#SBATCH --kill-on-invalid-dep=yes]

trap 'rm -f <script path>' EXIT
eval $(conda shell.bash hook)
conda init bash
source activate <environment>

set -eo pipefail
<one slf command>
```

The name leads with the job's submission index zero-padded to four digits, then joins the unit name, the job name, and
the specifier with hyphens. It replaces every character outside `A-Za-z0-9._-` with an underscore, and the script and
both logs carry it. The two conditional directives appear together whenever this submission holds an allocation for one
of the job's prerequisites, newly queued or adopted, while a prerequisite that already succeeded stays on the descriptor
and contributes no dependency. Every upstream identifier joins one `afterok:` prefix separated by colons, and the kill
directive turns an unsatisfiable dependency into a terminal state a status query can report. No other directive is
emitted, so a partition, account, QOS, node count, and GPU request are all unexpressible.

**Resource requests come from the job's own estimate.** `--cpus-per-task` takes the job's core weight and `--mem` its
`resident_mb`, rounded up to whole gigabytes and floored at one. That `resident_mb` adds the pages a job maps and its
shared library image to the anonymous `memory_mb` on which a local pool budgets. Both come from the model
`/job-planning` owns, while `--time` takes one figure for the whole submission,
`orchestration/remote.py::REMOTE_JOB_WALLTIME_MINUTES`, which is 480 minutes when the caller names none. **The
environment activation sits outside error checking**, since a conda hook exports shell state whose exit status says
nothing about whether the environment is usable. `set -eo pipefail` is therefore emitted after the preamble, and the
script removes itself through an exit trap so its exit status stays the work's. **Each script runs the same `slf`
command a local run would**, from the dispatch table both backends share. A local dispatch also caps each job's width at
what this machine's core budget supplies, so prove a pipeline on one session locally before a project-wide remote batch.

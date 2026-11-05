# Containers (Docker and Apptainer)

The images contain the Python package, VELEST, FMTOMO and the FMTOMO Extras, built with
`scripts/build_fortran.sh` on Debian 12 (gfortran 12, default VELEST array limits). The
executables live in `/opt/seisvel/bin/osv-tools` and `OSV_TOOLS_DIR` points there. The optional
geo extras (`cartopy`, `rasterio`, `pyvista`, `lasio`) and GMT are not included; use the
conda install if you need them.

## Docker

```bash
docker build -t seisvel .
docker run --rm seisvel doctor --quick          # health check
docker run --rm seisvel doctor                  # with the VELEST/fm3d smoke tests
docker run --rm seisvel --help
```

The entrypoint is `seisvel`, so arguments are `seisvel` subcommands. To run on a project, mount its
directory at `/work` (the working directory) and run as your own user so output files are
owned by you:

```bash
docker run --rm -u "$(id -u):$(id -g)" -v "$PWD/projects/myproj:/work" seisvel doctor --config config.yaml
docker run --rm -u "$(id -u):$(id -g)" -v "$PWD/projects/myproj:/work" seisvel <subcommand> --config config.yaml
```

Paths inside `config.yaml` must be valid inside the container (relative to `/work`, or under
another `-v host:container` mount, e.g. `-v /data/StationXML:/data/StationXML:ro`).
Use `--cpus`/`--memory` to limit resources; `project.n_jobs` should not exceed the CPUs given.
For an interactive shell: `docker run --rm -it --entrypoint bash -v "$PWD:/work" seisvel`.

The image has a `HEALTHCHECK` running `seisvel doctor --quick`. The VELEST limits are the build
defaults; to change them add `--velest-ieq N ...` to the `scripts/build_fortran.sh` line in the
`fortran` stage of the `Dockerfile`.

## Apptainer / Singularity (HPC)

Build on a machine with Apptainer (and internet), then copy the `.sif` to the cluster:

```bash
apptainer build seisvel.sif apptainer.def              # self-contained, from the repo root
# or convert the Docker image:
apptainer build seisvel.sif docker-daemon://seisvel:latest
```

```bash
apptainer run seisvel.sif doctor --quick
apptainer run --bind /scratch/myproj:/work --pwd /work seisvel.sif doctor --config config.yaml
apptainer exec seisvel.sif ls /opt/seisvel/bin/osv-tools
```

Apptainer maps your user and `$HOME` automatically, so no `-u` is needed. On clusters set
`APPTAINER_TMPDIR` to a large scratch disk when building, and bind the data directories you
need with `--bind`.

## Continuous integration

`.github/workflows/ci.yml` builds the Docker image and runs `docker run seisvel:ci doctor --quick`
on every push and pull request.

# DINO (DOT) CODES

Source files for the http://dino.codes website.

## Deploying

The deployment process works using Github Pages for hosting.

First we need to add a submodule under `public/` which is the files to be hosted
on Github Pages, we just run:

```bash
git submodule add -b master git@github.com:joaofcosta/joaofcosta.github.io.git public
```

After that's the done we can leverage the `deploy.sh` file present in the repository which
allows deployment of the website using Github Pages. We can use it by running:

```bash
./deploy.sh "COMMIT MESSAGE"
```

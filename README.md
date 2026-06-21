# XS-Leaks Wiki

## Build Process

### Build locally

1. Install the [Hugo Framework](https://gohugo.io/getting-started/installing/) **extended** version > 0.68
2. build custom repo
3. Run `hugo server --minify` in root directory 
4. Open your browser and go to http://localhost:1313 (or as indicated by hugo output)

### Generate static files

1. Run `hugo --buildDrafts`

## Automatic Deployment

This repository uses [Github Actions](https://github.com/features/actions) to automatically build and publish a static version of the XS-Leaks Wiki once a Pull Request is accepted.

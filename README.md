# Networking 4 Devs

This repository holds the sources for the network4dev.net website.

# Setting up locally

To develop locally and build the website as you edit the documentation, here's what you have to do. You will need a shell, `git` and `python3`.

```
# Create a local project folder
mkdir network4dev && cd network4dev

# First, clone this repo locally
git clone https://github.com/network4dev/n4d.git

# Next, create a virtualenv
python3 -m venv venv
source venv/bin/activate

# Update and install dependencies
pip install -U pip setuptools
pip install sphinx sphinx-autobuild sphinx_rtd_theme recommonmark

# Build a server locally
cd n4d
sphinx-autobuild . _build/html 
```

# Contributing

We use GitHub to host code, to track issues and feature requests, as well as accept pull requests. The process is explained here - [Github Flow](https://guides.github.com/introduction/flow/index.html) - and we're happy to help you get on your way to contributing to the project!

All commits to the master branch must be done via PRs which are reviewed and approved by the maintainers.

So fork the repository, make some changes and submit them to us as a PR!

Once the code makes it into the `master` branch, the webhooks ensure the RTD build triggers and is pushed to the live website over at <network4dev.net>(https://network4dev.net)!

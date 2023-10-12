# Yama CMS - Jekyll starter example

This is a starter/example Jekyll repository, configured to interface with Yama CMS.

Yama CMS will write .md files to ./pages & ./_posts.
Once you've wired up your repository with Yama CMS, delete the `./pages/` and `./_posts/` files and update the configuration to match your needs.

---

## Installation and usage

You can either run this repository directly if you have the Ruby tooling installed; or you can use [Earthly](https://docs.yama-cms.com/docs/guide/build-deploy-earthly) (think Dockerfiles + Makefiles) to run the needed tools inside containers.

Running directly (to install, see [Download Ruby](https://www.ruby-lang.org/en/downloads/)):
```bash
bundle install
bundle exec jekyll serve
```
Running via Earthly (to install, see [Earthly's Get Started](https://earthly.dev/get-earthly)):
```bash
earthly +dev
```
For more information on how to use Earthly, see [our YamaCMS specific documentation](https://docs.yama-cms.com/docs/guide/build-deploy-earthly/) or [the official Earthly documentation](https://docs.earthly.dev/basics).


***

## Editing this README

When you're ready to make this README your own, just edit this file. If you need inspiration, see [makeareadme.com](https://www.makeareadme.com/) for templates and ideas.

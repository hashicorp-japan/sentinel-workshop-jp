# Contributing
As the code basis in this repository have been purely developed for personal usecases, but your contributions are always welcome!
**Please carefully read this page to make the code review process go as smoothly as possible and to maximize the likelihood of your contribution being merged.**

## Bug Reports
For bug reports or requests [submit an issue](https://github.com/hashicorp-japan/sentinel-workshop-jp/issues).
With leveraging [GitHub features](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/about-issue-and-pull-request-templates#issue-templates), we have an issue template for reporting bugs.
Please feel free to use it for reducing your time.

## Pull Requests
The preferred way to contribute is to fork the [main repository](https://github.com/hashicorp-japan/sentinel-workshop-jp) on GitHub.
The brief procedures to fork this repository are described below, still we are also welcome if you would directly make PRs within this repository instead of forking.

1. Fork the [main repository](https://github.com/hashicorp-japan/sentinel-workshop-jp).  Click on the 'Fork' button near the top of the page.  This creates a copy of the code under your account on the GitHub server.

2. Clone this copy to your local disk:
```shell
$ git clone git@github.com:${YOUR_USERNAME}/sentinel-workshop-jp.git
$ cd sentinel-workshop-jp
```

3. Create a branch to hold your changes and start making changes.
```shell
$ git switch --create ${YOUR_BRANCH_NAME}
```

4. Work on this copy on your computer using Git to do the version control. When you're done editing, run the following to record your changes in Git:
```shell
$ git add ${MODIFIED_FILENAME} [ ${MODIFIED_FILENAME} ... ]
$ git commit
```

5. Push your changes to GitHub with:
```shell
$ git push -u origin ${YOUR_BRANCH_NAME}
```

6. Finally, go to the web page of your fork of the `sentinel-workshop-jp` repository and click 'Pull Request' to send your changes for review.

### GitHub Pull Requests Docs
If you are not familiar with pull requests, review the [pull request docs](https://help.github.com/articles/using-pull-requests/).

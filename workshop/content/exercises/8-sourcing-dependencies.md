# Sourcing dependencies

Introducing new and/or derivative software works in today's software engineering project landscape is frequently supported by dependencies.  As a developer we use tools like `git`, `curl` and `tar` to retrieve and unpack these dependencies; sometimes you have the additional burden of moving or renaming files whilst merging content from disparate sources into a single directory.  Wouldn't it be more convenient if we could declaratively state what should be in that directory?

Let's look at another tool within Carvel called [vendir](https://carvel.dev/vendir/).

To understand what it's capable of doing for you we might first take a look at the [specification](https://carvel.dev/vendir/docs/latest/vendir-spec/).  We can synchronize content from one or more sources to one or more directories.  And we have a variety of sources we can synchronize with, like: http, git, github-release, docker image, or helm chart to name a few.

While we could `git clone` a repository that contains [examples](https://github.com/vmware-tanzu/carvel-vendir/tree/develop/examples) for all the supported sources, instead we'll use `vendir` to sync a subset.

Take a peek at the configuration with:

```execute
cat config-step-5-sourcing-dependencies/vendir.yml
```

Note that this bit of configuration employs an `includePaths` filter to fetch a set of resources into a parent directory maintaining the original path from the git url.

Try it out

```execute
cd config-step-5-sourcing-dependencies
vendir sync 
```
> We are setting the current directory to process the configuration.  And the default configuration file name to process is `vendir.yml`.

Now inspect the contents of the `config-step-5-sourcing-dependencies` directory.

```execute
tree 
```

You should see something like:

```
config-step-5-sourcing-dependencies
├── vendir.lock.yml
├── vendir.yml
└── vendor
    ├── examples
    │   ├── git
    │   │   └── vendir.yml
    │   ├── github-release
    │   │   └── vendir.yml
    │   ├── helm-chart
    │   │   └── vendir.yml
    │   ├── http
    │   │   └── vendir.yml
    │   └── image
    │       └── vendir.yml
    ├── LICENSE
    └── NOTICE

7 directories, 9 files
```

Next we'll take some time to get familiar with a few variants of `vendir` configuration the you could employ.

What if you wanted to retrieve a few compressed resources (e.g., .zip files) from a AWS S3 bucket and have them automatically unpacked in destination directories?

Try this out

```execute
cd vendor/examples/http
vendir sync
```

Then

```execute
tree 
```

What if you wanted to retrieve the source for a specific release version of a Helm chart from a chart repository?

Try this out

```execute
cd ../helm-chart
vendir sync 
```

Then

```execute
tree 
cd ../../../..
```

Hopefully you've seen that `vendir` is a useful spec and tool for synchronizing content.  Its purpose really shines in situations where product teams may require and consume dependencies that are packaged and available in variety of formats.
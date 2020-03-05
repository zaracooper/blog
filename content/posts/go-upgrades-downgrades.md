---
title: "How to easily upgrade, downgrade, and migrate Go modules"
date: 2020-03-05T02:03:53+03:00
draft: false
author: "Zara Cooper"
summary: "A tutorial on performing stress-free Go module upgrades, downgrades, and migrations."
image: "go-upgrades-downgrades.jpg"
---

Go 1.14 is officially out and module support is now [production ready](https://blog.golang.org/go1.14). This is an opportune time to migrate to modules. With a large codebase, upgrading, downgrading, and migrating to modules may seem like a daunting task because so many changes have to be made to the code. In this tutorial, with the help of [this Recipe Puppy Go client module](https://github.com/zaracooper/recipepuppy), I'll walk through how to make module upgrades, downgrades, and migrations hassle-free. 

## What's covered: 
1. [Installing the `mod` tool](#installing-the-mod-tool)
2. [Upgrades and migrations](#upgrades-and-migrations)
3. [Downgrades](#downgrades)
4. [Upgrading dependencies](#upgrading-dependencies)
5. [Downgrading dependencies](#downgrading-dependencies)

##  Installing the `mod` tool
The [`mod` tool](https://github.com/marwan-at-work/mod) makes upgrades and downgrades simple and quick. To install it:

1. Set `$GOPATH` if you are working outside of `$GOPATH`. 
2. Add `$GOPATH\bin` to your `$PATH`.
3. Set `GO111MODULE` to `on`. Alternatively, you could set this value when installing the tool.
4. Run:
    ```bash
    go get github.com/marwan-at-work/mod
    ```

## Upgrades and migrations
When using Go modules, any backward compatibility breaking changes require a major version upgrade. Migrating to modules is considered a breaking change and requires an upgrade of the major version. Other backward-incompatible changes can include deleting and modifying exported types, functions, constants, methods, and variables [among other things](https://blog.merovius.de/2015/07/29/backwards-compatibility-in-go.html). It's important to have a clear picture of what backward-incompatible changes are to be made before deciding to upgrade.

In the case of the Recipe Puppy client, we're going to change some function signatures in [`v1`](https://github.com/zaracooper/recipepuppy/tree/v1.0.0) from:

```go
func FindRecipes(searchTerm string) ([]Recipe, error) {
    ...
}

func FindRecipesByIngredient(ingredient string) ([]Recipe, error) {
    ...
}
```

to this in the next version:

```go
// this is a breaking change because the function arguments have been modified
func FindRecipes(recipeTitles []string, page int) ([]Recipe, error) {
    ...
}

// this is a new feature
func FindRecipesWithIngredients(recipeTitles []string, ingredients []string, page int) ([]Recipe, error) {
    ...
}

// this is a breaking change because the function arguments have been modified
func FindRecipesByIngredients(ingredients []string, page int) ([]Recipe, error) {
    ...
}
```

To upgrade to a new version:

1. Identify your next major version eg. `v1` → `v2`, `v4` → `v5`. In this instance, I'm upgrading from `v1` to `v2`.
2. Create a space for the new changes. This is done to comply with [**semantic import versioning**](https://zaracooper.github.io/blog/posts/go-module-tidbits/#13-semantic-import-versioning). I'm using the **major subdirectory** method to release my `v2`+ module. So I'll create a new `v2` directory at the module root. If you are using the **major branch** method, create a new branch.

    ```bash
    mkdir v2
    ```

3. Copy Go source files and the go.mod file from the current version to the new directory. 

    ```bash
    cp *.go v2/
    cp go.mod v2/
    ```
4. Upgrade to the new version. The `mod` tool will change all module and import paths in the Go source files and the go.mod to reflect the new major version number. In this case, the import path `github.com/zaracopper/recipepuppy` in `v1` will change to `github.com/zaracooper/recipepuppy/v2` for `v2` in the files copied over.
    ```bash
    cd v2
    mod upgrade
    ```
5. Add the [compatibility breaking changes](https://github.com/zaracooper/recipepuppy/blob/v2.0.0/v2/api.go) to the source code. I added the aforementioned changes.

6. Write/modify [tests](https://github.com/zaracooper/recipepuppy/blob/v2.0.0/v2/api_test.go) to cover the breaking changes. Run the tests to make sure that they pass and that the code works as expected. 

7. Tag the release if everything is stable. If you'd like more room to experiment with and test the new changes, tag it as a pre-release version. Use the `-m` flag to provide an optional message describing the changes made to accompany your tag.
    ```bash
    git tag v2
    ```

8. Push the tag. 
    ```bash
    git push origin v2
    ```
 
Github releases are different from Git tags. Github treats your tag as the title of the equivalent release. If you're using Github, you can add the tag version on Github and even modify the release message.

![Tag names and tag versions](tag-display-github.png)

![Modify tag names and tag versions](edit-github.png)

## Downgrades
Module downgrades can be a good idea in some situations. For example, maybe a severe bug was part of a new tag or maybe a developer is having second thoughts about making compatibility breaking changes or a wrong tag was pushed to the origin repository. In most cases, however, downgrading a module would be a terrible idea because it may involve deleting a published tag and users may have already started consuming it. The best option when dealing with some of the aforementioned scenarios would be to publish a new tag or send out a patch. 

 That being said, if the tag hasn't yet been published or consumption of the new release is limited, then downgrading a module would be okay. Downgrading involves:

1. Deleting the tag locally if one was already created. 
    ```bash
    git tag -d v2
    ```

2. Downgrading the module. This will modify the module and import paths by removing the versions number from them or replacing it with the previous version number. For example, when downgrading the new `v2`, module paths will change from `github.com/zaracooper/recipepuppy/v2` to `github.com/zaracooper/recipepuppy`.
    ```bash
    mod downgrade
    ```

## Upgrading dependencies
New versions of dependencies are released constantly. In most cases, it's always advisable to upgrade to take advantage of new features and keep up with patches. In large codebases, it may be daunting to upgrade if a dependency is used in multiple places. It's even trickier when trying to upgrade more than one dependency at a time. Since it can get pretty complicated, some maintainers even opt for phased migrations when deciding to upgrade. An easy way to minimize the work involved is to change module and import paths in one fowl swoop. For example, to upgrade the `github.com/jarcoal/httpmock` test dependency when a `v2` version comes out, all I'd have to do is:

```bash
mod upgrade --mod-name=github.com/jarcoal/httpmock
```

## Downgrading dependencies
Sometimes using a new dependency may not work out and you may need to use a previous version that had worked better. A faster way to downgrade a dependency is to use `mod downgrade`. If I decided to downgrade `github.com/jarcoal/httpmock`, all I'd have to do is:

```bash
mod downgrade --mod-name=github.com/jarcoal/httpmock
```

## Conclusion
Module support has been a godsend. However, it can involve a ton of work when it comes to enforcing semantic import versioning when making and using `v2`+ modules. The `mod` command is an amazing tool that makes migrations, downgrades and upgrades faster and less taxing. I hope this was insightful.

## Credits
[Gophers](https://github.com/egonelbre/gophers)
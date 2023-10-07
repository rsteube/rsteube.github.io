# A pragmatic approach to shell completion

## The rise of new shells

One of the biggest improvements to my terminal experience was the switch from [Bash] to [Zsh].
The reason for this is the astonishing amount of completions provided by the community.
As well as something I didn't know of before: menu completion. A mode where you can cycle through the possible values. But like [vim] [Zsh] is quite tough on new users.

Then there was [Fish]. A shell that is more accessible to new users and thus calls itself friendly. But one major catch: the break from [POSIX].
This is actually a good thing yet the lack of completions at the time prevented me to make the switch.

Now there is a new generation of shells that allow passing structured data through a pipe. This is a major thing and well worth its [own article](https://medium.com/@browninfosecguy/powershell-pipeline-182ea25c622).

One of these is [Elvish], a shell written in Go that runs on Linux, BSDs, macOS, and Windows.
It is also the shell that managed to make me switch again.

But let's focus on another of its features: filtering of entries during menu completion.
And not just the values but the descriptions as well. You will understand later why this is important.

[Elvish] only has a few completions though so I am back at the same problem I had with [Fish].
This time however I've got some tools under my belt.

So let's change that...

## Command-line arguments

Arguments passed to your application are just that: simple words. How these are parsed is logically defined by your application. Which is the reason why we have such an inconsistent mess.

So for the sake of simplicity let's concentrate on the more prevalent [POSIX] (GNU) style.
Here we got shorthand flags `-s`, longhand flags `--long`, subcommands, and positional arguments.
Some flags take an argument `-a value`. Others accept an optional argument directly attached to the flag name `--flagname=optionalValue`.

You can refer to [Anatomy of a shell CLI](https://en.wikipedia.org/wiki/Command-line_interface#Anatomy_of_a_shell_CLI) for a more sophisticated description.

## How does shell completion work

The core concept is to split up the current command line (until the current cursor position) into words.
Depending on these the completion script returns possible values to replace the current word with.

```sh
command --flag1 positionalArg1 "positionalArg2 with space" <TAB>
# ['command', '--flag1', 'positionalArg1', 'positionalArg2 with space', '']
# positionalArg3
# Arg3
# thirdArg
```

The value to use is either determined by menu selection or filtering until there is only one.

```sh
command --flag1 positionalArg1 "positionalArg2 with space" thi<TAB>
# ['command', '--flag1', 'positionalArg1', 'positionalArg2 with space', 'thi']
# thirdArg
```

## What makes shell completion so complicated

The shell does not know the argument structure of your application.
So we have to tell it about subcommands, flags, and positional arguments.
But few provide a decent framework for this.

Then there are various caveats like special characters not being escaped.
These need to be addressed. For each application. For each shell. Over and over again.
In a scripting language, you are likely unfamiliar with and is hard to debug.

Completion scripts also tend to get quite [large and hard to read](https://github.com/git/git/blob/master/contrib/completion/git-completion.bash).

## A different approach to shell completion

One way to avoid writing the completion scripts by hand is to generate them. This is a feature provided by several argument parsers and ensures the scripts are kept in sync with your application.
But due to the difficulty of creating a generator, you can expect some issues:

- it supports only a few shells
- it is limited to flags and subcommands
- if there is support for value completion it only completes static values
  ```sh
  ["fixed", "words"] # static
  $(command generate values) # dynamic
  ```
- it only mimics your argument structure and subcommand determination is [rather optimistic](https://github.com/fish-shell/fish-shell/blob/master/share/functions/__fish_seen_subcommand_from.fish)
- you still end up with large scripts that are hard to debug
- inconsistencies between shells

A different approach is to only use the completion script to pass the words as arguments to your application.

```sh
command --flag1 positionalArg1 "positionalArg2 with space" <TAB>
# command complete command --flag1 positionalArg1 "positionalArg2 with space" ''
```

The argument parser then does what it is meant to do and the application returns the possible values.
This has several benefits:

- short scripts that are the same for each application
- easy testing of completion by invoking the application
- argument structure and subcommand determination is exact
- consistent behavior between shells
- completion possibilities that exceed what the shell provides
- adding new shells is quite simple

However, the startup time of the application matters as it affects the overall experience.

Now to the tools...

## carapace

[carapace] is a command argument completion generator based on [cobra].
It started as a mere fork of [#646](https://github.com/spf13/cobra/pull/646) with added dynamic completion while waiting for pull requests to be merged.
But development regarding this was a bit stale\* at the time and it ultimately became a project of its own.

It currently supports consistent dynamic completion for [Bash], [Elvish], [Fish], [Ion], [Nushell], [Oil], [Powershell], [Tcsh], [Xonsh], and [Zsh].

_(\*) Things have changed though and by now cobra has dynamic completion as well. It even took a similar approach so be sure to check that out._

## RawValue

[RawValue] is the internal representation of possible completion values.
It contains the actual value to be inserted, a display value to be shown during completion, and a description. It is important to split these as some shells show the description as a tooltip and to keep the display values short.

```sh
chown cups:<TAB>
# ['chown', 'cups:']
# {"value": "cups:audio", "display": "audio", "description":, "995"}
# {"value": "cups:root", "display": "root", "description":, "0"}
```

## Action

An [Action] indicates how to complete a flag or positional argument.
It contains either the static [RawValue]s or a function to generate them. Also some metadata like whether a space suffix shall be added.
None of this is public though and interaction with it is accomplished purely through functions.

One core function which pretty much all others end up using is [ActionValuesDescribed].
```go
ActionValuesDescribed(
  "audio", "995",
  "root", "0",
)
// {"value": "audio", "display": "audio", "description":, "995"}
// {"value": "root", "display": "root", "description":, "0"}
```

## Reusability

A great aspect of an [Action] is its reusability. Let's have a closer look at the `user:group` completion of [chown].

It uses [ActionMultiParts] which implicitly splits the current word (or part of it as these can be nested as well).
The parts themself are completed with [ActionUsers] and [ActionGroups]. Here the first one is invoked explicitly to add a `:` suffix to the inserted values.

```go
// ActionUserGroup completes system user:group separately
//   bin:audio
//   lp:list
func ActionUserGroup() carapace.Action {
	return carapace.ActionMultiParts(":", func(c carapace.Context) carapace.Action {
		switch len(c.Parts) {
		case 0:
			return ActionUsers().Invoke(c).Suffix(":").ToA()
		case 1:
			return ActionGroups()
		default:
			return carapace.ActionValues()
		}
	})
}
```

## Flexibility

The [chown] command generally takes `user:group` as first positional argument and files otherwise.
But here's the twist: when the `--reference` flag is used the first positional argument is also a file.
The benefit of an argument parser is that this can be checked pretty easily.

```go
rootCmd.Flags().String("reference", "", "use RFILE's owner and group rather than specifying OWNER:GROUP values")

carapace.Gen(rootCmd).FlagCompletion(carapace.ActionMap{
	"reference": carapace.ActionFiles(), // register flag argument completion
})

carapace.Gen(rootCmd).PositionalCompletion( // register completion for position 1..
	carapace.ActionCallback(func(c carapace.Context) carapace.Action {
		if rootCmd.Flag("reference").Changed { // check if --reference flag was used
			return carapace.ActionFiles()
		} else {
			return os.ActionUserGroup()
		}
	}),
)

carapace.Gen(rootCmd).PositionalAnyCompletion(
	carapace.ActionFiles(), // complete files for every other position
)
```

## Caching

The more dynamic the completions are the more apparent it becomes that caching is needed.
And while [GraphQL](https://graphql.org/) has proven to be great to keep transferred data minimal and thus delays are small it is good to avoid unnecessary internet roundtrips.

[Cache](https://rsteube.github.io/carapace/carapace/cache.html) wraps an [Action] and persists the generated [RawValue]s as json on disk. On the next invocation these are loaded unless the timeout has exceeded.

```go
func ActionIssueLabels(project string) carapace.Action {
    return carapace.ActionCallback(func(c carapace.Context) carapace.Action {
        // retrieve labels for project
    }).Cache(24*time.Hour, cache.String(project))
}
```

## Batch processing

Some [Action]s consist of several others. And invoking them sequentially can take a considerable amount of time.
[Batch](https://rsteube.github.io/carapace/carapace/batch.html) bundles these so that they can be invoked in parallel using goroutines.

The positional completion for the [docker inspect] command is such a case.

```go
carapace.Batch(
	docker.ActionContainers(),
	docker.ActionNetworks(),
	docker.ActionNodes().Supress("This node is not a swarm manager"),
	docker.ActionRepositoryTags(),
	docker.ActionSecrets().Supress("This node is not a swarm manager"),
	docker.ActionServices().Supress("This node is not a swarm manager"),
	docker.ActionVolumes(),
).ToA()
```

## carapace-bin

What originally was a sandbox for trying out [carapace] has evolved to [carapace-bin].
A command argument completer working across multiple shells and for multiple commands.
It can be seen as my personal collection of completions baked into a single binary.

One command I am using a lot is the [GitHub CLI].
Here, the importance of description filtering mentioned in the beginning becomes obvious.
E.g. the issue number on its own is not very helpful at finding the one you are looking for.
![](./a-pragmatic-approach-to-shell-completion/carapace-bin.cast)

## A sample application

Writing tools always felt very limited.
While it's quick to create a CLI nowadays, without shell completion it lacks usability.
[go-jira-cli](https://github.com/rsteube/go-jira-cli) is a quick take to see what is possible with [carapace].

It is a simple [Jira](https://www.atlassian.com/software/jira) terminal client composed of [carapace], [go-jira](https://github.com/andygrunwald/go-jira), and [GitHub CLI].
Here, flag completion has a strong dependency on other flag values. Also, notice the initial delay of the component flag which wasn't cached yet.

![](./a-pragmatic-approach-to-shell-completion/go-jira-cli.cast)


## lazycomplete

Commands are often sourced in shell startup scripts to register the completion.
```sh
source <(command completion)
```

But this is generally [discouraged](https://jzelinskie.com/posts/dont-recommend-sourcing-shell-completion/) because it slows down the shell startup. Invoking the command is not wrong per se though but rather a symptom of the shell doing this eagerly.

[lazycomplete](https://github.com/rsteube/lazycomplete) circumvents this problem by registering a temporary completion function that replaces itself on the first `<TAB>` with the actual one.
The initial delay is thus reduced to a single command invocation.
It is the same concept [carapace-bin] uses to register all of its completers in mere milliseconds.

```sh
source <(lazycomplete \
  example 'example _carapace' \
  lab 'lab completion' \
)

# _lazycomplete_example() {
#   unset -f _lazycomplete_example
#   source <(example _carapace)
#   $"$(complete -p example | awk '{print $3}')"
# }
# complete -F _lazycomplete_example example
# 
# _lazycomplete_lab() {
#   unset -f _lazycomplete_lab
#   source <(lab completion)
#   $"$(complete -p lab | awk '{print $3}')"
# }
# complete -F _lazycomplete_lab lab
```

## Afterword

All this is a limited view of the more easy part of shell completion though.
[pflag](https://github.com/spf13/pflag) is a flag parser for [POSIX] so completing commands not adhering to it quickly becomes quite hacky (at least for the moment).
But thanks to the high-level abstraction this can be improved without affecting the actual completers too much.

A great benefit is that I can now switch between shells without feeling out of place.
Shells become a specialized tool I can use not something I am stuck with.
I can open [Bash]/[Zsh] when I am copying a [POSIX] snippet from the internet.
Maybe [Xonsh] for something ML-related. Or [Nushell] to work with [large data](https://www.nushell.sh/book/dataframes.html).

So while it's certainly not without issues and still has some messy parts it works pretty well for me until someone comes up with a better solution.

[Action]:https://rsteube.github.io/carapace/carapace/action.html
[ActionGroups]:https://github.com/rsteube/carapace-bin/blob/0.8.1/pkg/actions/os/os.go#L34
[ActionUsers]:https://github.com/rsteube/carapace-bin/blob/0.8.1/pkg/actions/os/os.go#L168
[ActionValuesDescribed]:https://rsteube.github.io/carapace/carapace/action/actionValuesDescribed.html
[docker inspect]:https://github.com/rsteube/carapace-bin/blob/25e9706d444d979d15d35bc245bd81f47c686cf5/completers/docker_completer/cmd/inspect.go
[ActionMultiParts]:https://rsteube.github.io/carapace/carapace/action/actionMultiParts.html
[Bash]:https://www.gnu.org/software/bash/
[chown]:https://github.com/rsteube/carapace-bin/blob/0.8.1/completers/chown_completer/cmd/root.go
[Elvish]:https://elv.sh/
[Fish]:https://fishshell.com/
[Ion]:https://doc.redox-os.org/ion-manual/
[GitHub CLI]:https://cli.github.com/
[Nushell]:https://www.nushell.sh/
[Oil]:http://www.oilshell.org/
[POSIX]:https://en.wikipedia.org/wiki/POSIX
[Powershell]:https://microsoft.com/powershell
[RawValue]:https://pkg.go.dev/github.com/rsteube/carapace@v0.12.1/internal/common#RawValue
[Tcsh]:https://www.tcsh.org/
[Xonsh]:https://xon.sh/
[Zsh]:https://www.zsh.org/
[carapace-bin]:https://github.com/rsteube/carapace-bin
[carapace]:https://github.com/rsteube/carapace
[cobra]:https://github.com/spf13/cobra
[vim]:https://www.vim.org/

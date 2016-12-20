# BASH/ZSH auto-complete plugin for Composer

This is an experimental hack to add [Symfony BASH auto complete](https://github.com/stecman/symfony-console-completion) to Composer via a plugin. It's a pretty slimy hack, but it works without editing Composer's code.

![Composer BASH completion](https://i.imgur.com/MoDWkby.gif)

## Installation

1. Run `composer global require stecman/composer-bash-completion-plugin dev-master`
2. Add a completion hook to your shell's user config file:
  - If you're using BASH, put the following in your `~/.bash_profile` file:

    ```bash
    # Modified version of what `composer _completion -g -p composer` generates
    # Composer will only load plugins when a valid composer.json is in its working directory,
    # so  for this hack to work, we are always running the completion command in ~/.composer
    function _composercomplete {
        local CMDLINE_CONTENTS="$COMP_LINE"
        local CMDLINE_CURSOR_INDEX="$COMP_POINT"
        local CMDLINE_WORDBREAKS="$COMP_WORDBREAKS";

        export CMDLINE_CONTENTS CMDLINE_CURSOR_INDEX CMDLINE_WORDBREAKS

        # Honour the COMPOSER_HOME variable if set
        local composer_dir=$COMPOSER_HOME
        if [ -z "$composer_dir" ]; then
            composer_dir=$HOME/.composer
        fi

        local RESULT STATUS;

        RESULT=$(cd $composer_dir && composer depends _completion);
        STATUS=$?;

        local cur;
        _get_comp_words_by_ref -n : cur;

        if [ $STATUS -ne 0 ]; then
            echo -e "$RESULT";
            return $?;
        fi;

        COMPREPLY=(`compgen -W "$RESULT" -- $cur`);

        __ltrim_colon_completions "$cur";
    };

    complete -F _composercomplete composer;
    ```
  - If you're using ZSH, put the following in your `~/.zshrc` file:
    
    ```bash
    function _composer {
        local -x CMDLINE_CONTENTS="$words"
        local -x CMDLINE_CURSOR_INDEX
        (( CMDLINE_CURSOR_INDEX = ${#${(j. .)words[1,CURRENT]}} ))

        # Honour the COMPOSER_HOME variable if set
        local composer_dir=$COMPOSER_HOME
        if [ -z "$composer_dir" ]; then
            composer_dir=$HOME/.composer
        fi

        local RESULT STATUS
        RESULT=("${(@f)$( cd $composer_dir && composer depends _completion )}")
        STATUS=$?;

        if [ $STATUS -ne 0 ]; then
            echo -e "$RESULT";
            return $?;
        fi;

        compadd -- $RESULT
    };

    compdef _composer composer;
    ```
3. Reload the modified shell config (or open a new shell), and enjoy tab completion on Composer

## Explanation

This hacky plugin injects an additional command into the Composer application at runtime. When the plugin in this package is activated and the command line starts with `composer depends _completion`, the plugin effectively reboots the application with the completion command added, and drops `depends` from the command line so that `_completion` becomes the command argument. This used to work without piggy-backing on a command, but an update to composer stopped the original method working (#8).

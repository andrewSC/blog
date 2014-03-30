---
layout: post
title: "Git autocompletion and bashthings"
date: 2013-02-16 14:12
comments: false
categories:
---

{% img http://i.imgur.com/7LhFQ36.png %}

At my work we use Git as a VCS and while Git is very powerful, trying
to remember all those little arguments can become tedious at best.  Great, we established
aliases, autocompletion, and Git alias autocompletion would be nice but how do we go about setting all that up?

The first thing you're going to want to do is setup/install the [git-completion.bash](http://git-scm.com/book/en/Git-Basics-Tips-and-Tricks#Auto-Completion) script

Next, add these entries into your .bash_profile (for OS X) or .bashrc (for Linux)
{% codeblock lang:sh %}
BASH_GIT_COMPLETION="/usr/local/etc/bash_completion.d/git-completion.bash"
GIT_PS1_SHOWDIRTYSTATE="true"
GIT_PS1_SHOWUPSTREAM="auto"
#bash completion
if [ -f `brew --prefix`/etc/bash_completion ]; then
     . `brew --prefix`/etc/bash_completion
fi
{% endcodeblock %}

Note: the `BASH_GIT_COMPLETION` path will vary depending on your setup.  As a note, these are my git aliases.  Feel free to use them:
{% codeblock lang:sh %}
alias g='git'
alias ga='git add'
alias gaa='git add -A'
alias gb='git branch'
alias gba='git branch -a'
alias gbam='git add . && git commit -am'
alias gc='git commit -v'
alias gca='git commit -v -a'
alias gcam='git commit -am'
alias gcm='git commit -m'
alias gco='git checkout'
alias gcount='git shortlog -sn'
alias gcp='git cherry-pick'
alias gd='git diff'
alias gdf='git diff --full-index master'
alias gf='git fetch'
alias gl='git log --graph --pretty=format:"%C(yellow)%h%Cred%d\\ %Creset%s%Cblue\\ [%cn]" --decorate'
alias glg='git log --stat --max-count=5'
alias glgg='git log --graph --max-count=5'
alias glp='git log --oneline --decorate'
alias gmt='git mergetool -t diffmerge'
alias gpo='git pull origin'
alias grh='git reset HEAD'
alias grhh='git reset HEAD --hard'
alias grm='git status | grep deleted | awk '\''{print $3}'\'' | xargs git rm'
alias gs='git status'
alias gsd='git svn dcommit'
alias gsr='git svn rebase'
alias gss='git status -s'
alias gsu='git submodule'
alias gup='git fetch && git rebase'
{% endcodeblock %}


After you've added/source'd the aliases file into your `.bash_profile` you can insert/source the `alias_completion` script
{% codeblock alias_completion lang:sh http://superuser.com/questions/436314/how-can-i-get-bash-to-perform-tab-completion-for-my-aliases Source %}
function alias_completion {
local namespace="alias_completion"
local compl_regex='complete( +[^ ]+)* -F ([^ ]+) ("[^"]+"|[^ ]+)'
local alias_regex="alias ([^=]+)='(\"[^\"]+\"|[^ ]+)(( +[^ ]+)*)'"
eval "local completions=($(complete -p | sed -Ene "/$compl_regex/s//'\3'/p"))"

(( ${#completions[@]} == 0 )) && return 0

rm -f "/tmp/${namespace}-*.tmp" # preliminary cleanup
local tmp_file="$(mktemp "/tmp/${namespace}-${RANDOM}.tmp")" || return 1

local line; while read line; do
eval "local alias_tokens=($line)" 2>/dev/null || continue # some alias arg patterns cause an eval parse error
local alias_name="${alias_tokens[0]}" alias_cmd="${alias_tokens[1]}" alias_args="${alias_tokens[2]# }"

eval "local alias_arg_words=($alias_args)" 2>/dev/null || continue

[[ " ${completions[*]} " =~ " $alias_cmd " ]] || continue
local new_completion="$(complete -p "$alias_cmd")"

# create a wrapper inserting the alias arguments if any
if [[ -n $alias_args ]]; then
	local compl_func="${new_completion/#* -F /}"; compl_func="${compl_func%% *}"
	# avoid recursive call loops by ignoring our own functions
	if [[ "${compl_func#_$namespace::}" == $compl_func ]]; then
		local compl_wrapper="_${namespace}::${alias_name}"
		echo "function $compl_wrapper {
		(( COMP_CWORD += ${#alias_arg_words[@]} ))
		COMP_WORDS=($alias_cmd $alias_args \${COMP_WORDS[@]:1})
		$compl_func
	}" >> "$tmp_file"
	new_completion="${new_completion/ -F $compl_func / -F $compl_wrapper }"
fi
  fi
  new_completion="${new_completion% *} $alias_name"
  echo "$new_completion" >> "$tmp_file"
done < <(alias -p | sed -Ene "s/$alias_regex/\1 '\2' '\3'/p")
source "$tmp_file" && rm -f "$tmp_file"
}; alias_completion
{% endcodeblock %}

Add PS1 prompt gaudiness
{% codeblock PS1 prompt lang:sh https://github.com/necolas/dotfiles Source %}
if [[ $COLORTERM = gnome-* && $TERM = xterm ]] && infocmp gnome-256color >/dev/null 2>&1; then
	export TERM=gnome-256color
elif infocmp xterm-256color >/dev/null 2>&1; then
	export TERM=screen-256color
fi
tput sgr 0 0

BOLD=$(tput bold)
RESET=$(tput sgr0)
SOLAR_YELLOW=$(tput setaf 136)
SOLAR_ORANGE=$(tput setaf 166)
SOLAR_RED=$(tput setaf 124)
SOLAR_MAGENTA=$(tput setaf 125)
SOLAR_VIOLET=$(tput setaf 61)
SOLAR_BLUE=$(tput setaf 33)
SOLAR_CYAN=$(tput setaf 37)
SOLAR_GREEN=$(tput setaf 64)
SOLAR_WHITE=$(tput setaf 254)

style_user="\[${RESET}${SOLAR_ORANGE}\]"
style_host="\[${RESET}${SOLAR_YELLOW}\]"
style_path="\[${RESET}${SOLAR_GREEN}\]"
style_chars="\[${RESET}${SOLAR_WHITE}\]"
style_branch="${SOLAR_CYAN}"

if [[ "$SSH_TTY" ]]; then
	# connected via ssh
	style_host="\[${BOLD}${SOLAR_RED}\]"
elif [[ "$USER" == "root" ]]; then
	# logged in as root
	style_user="\[${BOLD}${SOLAR_RED}\]"
fi

is_git_repo() {
	$(git rev-parse --is-inside-work-tree &> /dev/null)
}

is_git_dir() {
	$(git rev-parse --is-inside-git-dir 2> /dev/null)
}

get_git_branch() {
	local branch_name

	# Get the short symbolic ref
	branch_name=$(git symbolic-ref --quiet --short HEAD 2> /dev/null) ||
		# If HEAD isn't a symbolic ref, get the short SHA
	branch_name=$(git rev-parse --short HEAD 2> /dev/null) ||
		# Otherwise, just give up
	branch_name="(unknown)"

	printf $branch_name
}

# Git status information
prompt_git() {
	local git_info git_state uc us ut st

	if ! is_git_repo || is_git_dir; then
		return 1
	fi

	git_info=$(get_git_branch)

	# Check for uncommitted changes in the index
	if ! $(git diff --quiet --ignore-submodules --cached); then
		uc="+"
	fi

	# Check for unstaged changes
	if ! $(git diff-files --quiet --ignore-submodules --); then
		us="!"
	fi

	# Check for untracked files
	if [ -n "$(git ls-files --others --exclude-standard)" ]; then
		ut="?"
	fi

	# Check for stashed files
	if $(git rev-parse --verify refs/stash &>/dev/null); then
		st="$"
	fi

	git_state=$uc$us$ut$st

	# Combine the branch name and state information
	if [[ $git_state ]]; then
		git_info="$git_info[$git_state]"
	fi

	printf "${SOLAR_WHITE} on ${style_branch}${git_info}"
}

# Set the terminal title to the current working directory
PS1="\[\033]0;\w\007\]"
# Build the prompt
PS1+="\n" # Newline
PS1+="${style_user}\u" # Username
PS1+="${style_chars}@" # @
PS1+="${style_host}\h" # Host
PS1+="${style_chars}: " # :
PS1+="${style_path}\w" # Working directory
PS1+="\$(prompt_git)" # Git details
PS1+="\n" # Newline
PS1+="${style_chars}\$ \[${RESET}\]" # $ (and reset color)
{% endcodeblock %}

Finally, man page stuff
{% codeblock lang:sh %}
# Setting colorized man pages
export LESS_TERMCAP_mb=$(printf '\033[38;5;92m') # enter blinking mode - red
export LESS_TERMCAP_md=$(printf '\033[38;5;84m') # enter double-bright mode - bold, magenta
export LESS_TERMCAP_me=$(printf '\e[0m') # turn off all appearance modes (mb, md, so, us)
export LESS_TERMCAP_se=$(printf '\e[0m') # leave standout mode
export LESS_TERMCAP_so=$(printf '\033[38;5;196m') # enter standout mode - yellow
export LESS_TERMCAP_ue=$(printf '\e[0m') # leave underline mode
export LESS_TERMCAP_us=$(printf '\033[38;5;208m') # enter underline mode - cyan
{% endcodeblock %}

Latest hotness? Check out the github repo: [https://github.com/andrewSC/bash_profile](https://github.com/andrewSC/bash_profile) (´◔◞౪◟◔)


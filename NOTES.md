# go_lab

## add git branch to bash profile
add to your .bashrc file

```
parse_git_branch() {
  git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
}

export PS1="\u@\h \W\[\033[32m\]\$(parse_git_branch)\[\033[00m\] $ "
```
run this command after you finish the edit
```
source ~/.bashrc
```
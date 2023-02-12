# jgreat.github.io

## Local Development

This is sorta the dirty install.  I hate that ruby wants to do everything at the system level.
Should make this a docker install.

### Install Jekyll

```
# Install OS packages
sudo apt-get install ruby-full build-essential zlib1g-dev

# Add gem path to zshrc
echo '# Install Ruby Gems to ~/gems' >> ~/.zshrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.zshrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Install jekyll/bundler
gem install jekyll bundler
```

### Testing

```
# Install/Update deps
bundle install

# Start dev server
bundle exec jekyll serve
```

### Add additional dependencies

```
bundle add webrick
```

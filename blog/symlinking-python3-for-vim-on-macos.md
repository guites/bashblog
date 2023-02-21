Symlinking python3 for vim on macOS

Managing binaries in macos can be a shitshow, even with the assistance of great projects like [brew](https://brew.sh/).

One of brew's [most](https://github.com/Homebrew/brew/issues/9285) [debatable](https://docs.brew.sh/FAQ#why-does-brew-upgrade-formula-or-brew-install-formula-also-upgrade-a-bunch-of-other-stuff) feature is that installing a new package will, by default, trigger an automatic update of all existing packages. And this break things.

By installing some package I don't really remember the name, I ended up updating my python3 version, or rather, installing a new one, Python 3.11 .

This broke my beloved vim workflow :'( which uses [Black](https://github.com/psf/black) for formatting (by running `:Black` inside vim) 

So if you're on macOS running into the following error when running `:Black` from inside vim:

        Error detected while processing /Users/guilhermegarcia/.vim/plugged/black/autoload/black.vim:
        line  214:
        Traceback (most recent call last):
          File "<string>", line 113, in <module>
          File "<string>", line 89, in _initialize_black_env
          File "/opt/homebrew/Cellar/python@3.11/3.11.1/Frameworks/Python.framework/Versions/3.11/lib/python3.11
        /venv/__init__.py", line 468, in create
            builder.create(env_dir)
          File "/opt/homebrew/Cellar/python@3.11/3.11.1/Frameworks/Python.framework/Versions/3.11/lib/python3.11
        /venv/__init__.py", line 74, in create
            self.setup_python(context)
          File "/opt/homebrew/Cellar/python@3.11/3.11.1/Frameworks/Python.framework/Versions/3.11/lib/python3.11
        /venv/__init__.py", line 292, in setup_python
            copier(context.executable, path)
          File "/opt/homebrew/Cellar/python@3.11/3.11.1/Frameworks/Python.framework/Versions/3.11/lib/python3.11
        /venv/__init__.py", line 235, in symlink_or_copy
            shutil.copyfile(src, dst)
          File "/opt/homebrew/Cellar/python@3.11/3.11.1/Frameworks/Python.framework/Versions/3.11/lib/python3.11
        /shutil.py", line 256, in copyfile
            with open(src, 'rb') as fsrc:
                 ^^^^^^^^^^^^^^^
        FileNotFoundError: [Errno 2] No such file or directory: '/opt/homebrew/opt/python@3.11/Frameworks/Python
        .framework/Versions/3.11/bin/python3'
        Error detected while processing function black#Black:
        line   10:
        Traceback (most recent call last):
          File "<string>", line 1, in <module>
        NameError: name 'Black' is not defined
        Press ENTER or type command to continue

Turns out that somewhere, the psf/black plugin is trying to access a python binary that just isn't there:

        guites@macos ~ % ls -l /opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/bin/python3
        ls: /opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/bin/python3: No such file or directory

It is actually at `/opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/bin/python3.11` (notice the `.11`)

So the most straightforward fix I've found is just creating a link between the non existing binary name and the correct python installation:

        ln /opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/bin/python3.11 /opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/bin/python3

And from there on, it should just work. Inside vim:

        :Black

        Please wait, one time setup for Black.

        Creating a virtualenv in /Users/guilhermegarcia/.vim/black...
        (this path can be customized in .vimrc by setting g:black_virtualenv)
        Installing Black with pip...
        DONE! You are all set, thanks for waiting ✨ 🍰 ✨
        Pro-tip: to upgrade Black in the future, use the :BlackUpgrade command and restart Vim.
        Black: already well formatted, good job. (took 0.0057s)
        Press ENTER or type command to continue

Neat! You can safely remove the symlink (`unlink /opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/bin/python3`) after this step, but its not necessary.

For reference, I'm running vim 9.0 with [vim-plug](https://github.com/junegunn/vim-plug), and installed Black by adding the following lines to my .vimrc:

        call plug#begin('~/.vim/plugged')

        Plug 'psf/black', { 'branch': 'stable' }

        call plug#end()

Some other links that didn't solve my problem:

- <https://github.com/psf/black/issues/2876>
- <https://github.com/psf/black/issues/2503>
- <https://github.com/psf/black/issues/2547>
- <https://github.com/psf/black/issues/1379>
- <https://stackoverflow.com/questions/36001966/unable-to-run-python-3-after-homebrew-installation>
- <https://www.digitalocean.com/community/tutorials/how-to-fix-python-no-such-file-or-directory-compiler-errors-when-installing-packages>

Tags: vim, cli, python, macos

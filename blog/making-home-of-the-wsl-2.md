Making home of the WSL 2

Having recently migrated from Ubuntu to Windows as my work machine, many of the features that made my system feel like home were missing.

This post serves as a collection of solutions I found \(and some I've adapted\) from the web. Cheers!

<hr/>

## Correctly installing Docker under WSL

You do not \(and should not\) install anything Docker related inside your WSL instances. Instead, you can use the Docker Desktop WSL 2 Backend, which will take care of integrating both systems, and running Docker both in your Windows and in your WSL instances.

Make sure you have WSL 2 installed. You can check, in PowerShell, running

		wsl -l -v

If you see output like this

		PS C:\Users\guites> wsl -l -v
		  NAME                   STATE           VERSION
		* Ubuntu-22.04           Running         2

It means you are all set. If your `version` column shows only version 1, you can [follow the official documentation](https://learn.microsoft.com/en-us/windows/wsl/install#upgrade-version-from-wsl-1-to-wsl-2) to upgrade your wsl from version 1 to 2.

Then, you need to change the version used by your distro \(in powershell\):

		wsl --set-version Ubuntu-22.04 2

Now, if you have previously installed docker directly from inside wsl, using for example the [docker engine for ubuntu documentation](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository), I strongly recommend uninstalling it completely.

[This post](https://docs.docker.com/desktop/windows/wsl/) goes into great detail about how you can completely remove old docker versions from wsl.

I found the following commands to be the only ones necessary \(inside WSL\):

		sudo apt-get remove docker docker-engine docker.io containerd runc
		sudo rm -rf /var/lib/docker /etc/docker /etc/apparmor.d/docker /var/run/docker.sock /usr/local/bin/docker-compose /etc/docker
		sudo groupdel docker

After removing Docker from WSL \(or in case you didn't have it installed in the first place\), make sure to uninstall Docker from your Windows also.

After that, proceed with the instalation of [Docker Desktop WSL 2 backend on Windows](https://docs.docker.com/desktop/windows/wsl/).

Docker will create two distinct WSL containers, that can be seems running the following command from PowerShell:

		wsl -l -v


		PS C:\Users\guites> wsl -l -v
		  NAME                   STATE           VERSION
		* Ubuntu-22.04           Running         2
		  docker-desktop         Running         2
		  docker-desktop-data    Running         2

They will be responsible for managing the communication between windows and wsl. You do not need to install anything Docker related inside of WSL.

## Acessing localhost from within a docker container

If you have a docker container inside WSL running some application on localhost, and want to access it from another docker container inside WSL, you won't be able to reach it from `http://localhost`.

You will have to use your machine IP. You can find it by running

	ip addr | grep eth0

	: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
	    inet 174.86.26.335/20 brd 174.86.25.255 scope global eth0
		^-------------^ ====> use this IP instead of localhost

Inside wsl. \([source](https://superuser.com/a/1642352)\)

## Copying files to the clipboard

You will not be able to run `xclip` or `xsel`, as you would in regular ubuntu.

Instead, you can access the `.exe` file responsible for copying in windows directly.

To copy the contents of a file `list-of-things.txt`

		cat list-of-things.txt | clip.exe

In this situations I like to create an alias so that I can call the same command name on both my linux and windows machine.

		echo "alias ,copy='clip.exe'" >> ~/.bash_aliases
		source ~/.bash_aliases

Now, you can copy the output of any command by running, for example

		ls -l | ,copy

And hit <kbd>ctrl + v</kbd> where you need it.

## Opening files and the browser from WSL terminal

There are many possible solutions, but the most straightforward seems to be creating an alias pointing to your browser and explorer executables.

**For the browser**, you first need to find where the application binary lives in your Windows machine.

For firefox, you can do this by opening a new page, going to `about:support` and scrolling down to `Application binary` \(right below `Operational System`\).

Mine is `C:\Program Files\Mozilla Firefox\firefox.exe`. But you must make a few changes to that path.

1. Change backslashes into forward slashes: `C:/Program Files/Mozilla Firefox/firefox.exe`
2. Remove the `:` from the path: `C/Program Files/Mozilla Firefox/firefox.exe`
3. Escape the spaces: `C/Program\ Files/Mozilla\ Firefox/firefox.exe`
4. Lowercase that C: `c/Program\ Files/Mozilla\ Firefox/firefox.exe`
5. \(First check if you need it\) append /mnt to the path: `/mnt/c/Program\ Files/Mozilla\ Firefox/firefox.exe`

Check that you can open it by running it in the WSL terminal:

		/mnt/c/Program\ Files/Mozilla\ Firefox/firefox.exe`

If everything goes well, create an alias

		echo "alias ,firefox='/mnt/c/Program\ Files/Mozilla\ Firefox/firefox.exe'" >> ~/.bash_aliases
		source ~/.bash_aliases

Test it by opening up an URL

		,firefox https://guilhermegarcia.dev/blog

**For the files**, you can simply use `explorer.exe`. In regular Ubuntu, you would use something like `xdg-open` or simply `open`, but the environment is not fully configured on the default WSL installation. Instead, lets create an alias!

		echo "alias ,open='explorer.exe'" >> ~/.bash_aliases
		source ~/.bash_aliases

And test it by opening some file

		echo "Contents" > file.txt
		,open file

You should see your friendly neighborhood notepad pop up :P.

[source](https://stackoverflow.com/questions/52691835/wsl-ubuntu-how-to-open-localhost-in-browser-from-bash-terminal)

## Disable the beep

That goddamn beep ( ` ω ´ ).

You can disable it **from inside WSL** by adding a `set bell-style none` in the `/etc/inputrc` file. \([source](https://stackoverflow.com/a/36726662)\)

Note that if you use vim, you will also have to add the following to your ~/.vimrc

		set visualbell
		set t_vb=

You can also disable it **from outside WSL** \(as a brute force solution\) by openning the Volme Mixer in Windows and muting each WSL terminal window.

## Conclusion

These are the adjustments I've made to my WSL so far. If you have any additional tips or comments, please send them to my [email address](mailto:gui.garcia67@gmail.com). Thanks!

Tags: windows, wsl

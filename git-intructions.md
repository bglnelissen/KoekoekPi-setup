# install git on the pi
*slightly modified from: http://www.instructables.com/id/GitPi-A-Private-Git-Server-on-Raspberry-Pi*

    sudo apt-get install --yes wget git-core

- nu een map maken die we gaan gebruiken als git 'host' (git is een gedecentraliseerd systeem, eigenlijk bestaat er niet zoiets als 'host').
- git init --bare also creates a repository, but it does not have the working directory. This means that you can not edit files, commit your changes, add new files in that repository. A default Git repository assumes that you will be using it as your working directory. ... A non-bare repository is the default.
- git 'host' directory voor mijn ~/bin folder:

```
mkdir ~/git/bin.git
cd ~/git/bin.git
git init --bare
```

- De pi is de server. Op de remote voegen we de server toe. We gebruiken de alias KoekoekPi.
- Op de remote gaan we naar de juiste map en initialiseren git voor die map en voor de server.

```
cd ~/bin
git init
git remote add KoekoekPi bas@guu.st:/home/bas/git/bin.git
```

Push code naar de (--bare) server:

```
git add .
git commit -m "initial commit"
git push KoekoekPi master
```

Clone the remote repo on another machine

```
cd ~/git/ # cd to where the project directory needs to be
git clone bas@guu.st:git/bin.git
```


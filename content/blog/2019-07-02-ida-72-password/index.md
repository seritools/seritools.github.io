+++
title = "How IDA 7.2's installer password was found"
+++

In January 2019, the installer files for IDA 7.2 were leaked. This does not mean it was usable however, as you need an installer password to install and a licence file to activate it.
Separately to that, a license file from ESET was leaked, which didn't match the feature set of the installer file.

But all the leaks didn't matter, because without the installer password, the program files were safe. Until now. :)

On 2019-06-21, devcore published a [blog post](https://devco.re/blog/2019/06/21/operation-crack-hacking-IDA-Pro-installer-PRNG-from-an-unusual-way-en/) about obvious flaws in the MacOS and Linux installers for IDA, including the password as plaintext in the setup file. The Windows installer, however, uses [InnoSetup](http://www.jrsoftware.org/isinfo.php) as installation engine.

InnoSetup encrypts the program data with the installer password and hashes it via SHA-1, prepending it with `PasswordCheckHash` and eight random bytes as salt. The password being 12 alphanumeric characters long means that bruteforcing it is pretty much out of the question.

Unless you find out how the passwords were generated in the first place! Devcore found out that the passwords are simply generated with a small perl script using `srand()`/`rand()`.

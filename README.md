# debcomp

DebComp (**Deb**ian **Comp**oser) is a configuration deployment system for Debian-based operating systems.

*In other languages: [English](README.md), [Русский](README.ru.md).*

---

## Table of Contents

- [Introduction](#introduction)
- [Key features](#key-features)
- [Configuration directory structure](#configuration-directory-structure)
- [Environment variables](#environment-variables)
- [Component configuration](#component-configuration)


## Introduction

In most cases, operating systems can be used with minor changes to their initial configuration. But some highly specific use cases require the creation of complex configurations that involve installing and configuring a large number of different software and making a large number of changes to the system configuration. Over time, their number grows and there is a need to systematize them, and re-making changes manually on a newly installed copy of the system becomes too laborious and not feasible in a reasonable time. At first glance, the problem is solved by simply copying the image of the system partition or its files to any other computers. But in any case, sooner or later there will be a situation when you have to repeat from scratch all the previously performed configuration, for example, if it is impossible to update the distribution or you need to install the system on a computer with a different processor architecture. A partial solution is to keep a record of all changes made in order to make them completely identical to other copies of the system in the future. But such a manual approach is too long, laborious and unreliable — there is always a risk that the human factor will play a role and something will be overlooked, a typo will be made somewhere and so on. There are slightly more successful solutions, such as automating the deployment of configurations using scripts, configuration management systems like Ansible, and so on. But they are far from perfect, since this is not always possible to rollback changes, do not allow them to be properly organized and do not contribute to simplifying the migration of configurations between different distributions and their versions.

The second problem is the current tendency to create many new OS distributions, most often based on Ubuntu or Debian, adapted for any highly specific tasks to the detriment of versatility. However, it is much more correct to create and distribute configurations for existing distributions instead of creating new distributions, since the software must solve specific tasks, and the operating system must ensure the operation of the application software, but not solve its tasks. This approach would allow the user to use their favorite distribution, rather than migrate to another, untested and often limited in capabilities. But without a configuration deployment tool with an easy-to-use user interface, this approach is not feasible in practice.

In order to solve the above problems, the DebComp project was created.


## Key features

- It is a portable script written in POSIX shell, with no dependencies, except for coreutils, which is an essential package in almost all distributions. This reduces configuration deployment to copying the configuration directory to the target system (for example, using scp) and executing debcomp without the need to install additional software;
- Presence of a text-based user interface (TUI);
- Ability to rollback system configuration changes (except for packages state);
- Flexible and transparent architecture, consistent with the UNIX philosophy and not overloaded with unnecessary elements;
- Combining imperative and declarative approaches, allowing you to flexibly interact with the OS directly from scripts, have a accurate understanding of all the actions performed and, if necessary, fine-adjust them, instead of using ready-made modules like a "black boxes", completely entrusting the task to them in the hope that everything will be done correctly and without errors, while the OS provides simple, flexible and reliable tools for its configuration without any unnecessary intermediate layers.


## Configuration directory structure

File or directory | Description
--- | ---
`common/` | A base component that contains a common part of the configuration.
`common/unmanaged-packages` | A file containing regular expressions for package names that should be ignored unless the package name is explicitly specified in the package list. These packages will be protected from removing and changing the mark from manual to auto (see the description of `pkglists/`), and its absence in the `for()` list will not be considered an error. Useful for packages whose names may change at any time (such as linux-image-*, which contains its version in the package name) and packages providing hardware support. In most cases is enough one regular expression <code>^linux(-.+)?-(generic&#124;headers&#124;image&#124;modules)(-.+)?$</code>, which prevents the removal of packages related to Linux kernels.
`components/` | A directory that contains the configuration components.
`{common,components/<component>}/files/` | A directory that contains files and directories for installation to the system. Before installation, the permissions for all directories and files are set to 755 and 644, respectively, and their owners are set to root:root. Changing permissions and owners must be done from the `preinst` script (example: `chmod 600 "$PREPDIR/etc/hostapd/hostapd.conf"`).
`{common,components/<component>}/pkglists/` | A directory containing files for lists of installed packages and their marks (manual or auto). It can contain subdirectories with any level of nesting. Package lists are generated using the `analyze-deps` subcommand (run `debcomp analyze-deps --help` command for details) and contains only packages that are in recommended, suggested, or selective dependencies (see https://www.debian.org/doc/debian-policy/ch-relationships.html) and packages marked as manual (i.e. needed by themselves, not just to satisfy the dependencies of other packages, see https://manpages.debian.org/stable/apt/apt-mark.8.en.html). When selecting a component, all packages listed in these files and their hard dependencies (listed in the Depends field) are installed, and the remaining packages are removed via the purge method.
`{common,components/<component>}/pkgs/` | A directory containing packages that will be installed together with the component, which can be built or unpacked for automatic building. Before building a package, the permissions for all its directories and files are set to 755 and 644, respectively, and their owners are set to root:root. Changing permissions and owners must be done from the `pkgs/<package>/DEBIAN/prebuild` script (example: `chmod 600 "$PREPDIR/etc/hostapd/hostapd.conf"`).
`{common,components/<component>}/apt-sources` | A file containing additional repositories to add to `/etc/apt/sources.list`.
`{common,components/<component>}/control` | Component configuration file. See &laquo;[Component configuration](#component-configuration)&raquo;.
`{common,components/<component>}/preinst` | A script that executes before installing files to the system.
`{common,components/<component>}/postinst` | A script that executes after installing files to the system.
`{common,components/<component>}/prerm` | A script that executes before removing files from the system (when rolling back the previously installed configuration).
`{common,components/<component>}/postrm` | A script that executes after removing files from the system (when rolling back the previously installed configuration).
`presets/` | A directory containing configuration presets of the components to be installed.
`config` | A file containing configuration of the components to be installed.
`debcomp` | Main executable file.


## Environment variables

Variable name | Available in scripts | Description
--- | --- | ---
`CONFDIR` | `preinst`, `postinst`, `prerm`, `postrm`, `prebuild` | Absolute path to the configuration directory.
`SCRIPT_NAME` | `preinst`, `postinst`, `prerm`, `postrm`, `prebuild` | The name of the current caused by the script.
`COMPONENT` | `preinst`, `postinst`, `prerm`, `postrm` | Path to the component directory relative to the `CONFDIR` directory.
`PACKAGE` | `prebuild` | Path to the package directory relative to the `CONFDIR` directory.
`PREPDIR` | `preinst`, `prebuild` | Path to the directory containing the files prepared for installation to the system or for building the package.
`TMPDIR` | `preinst`, `postinst`, `prerm`, `postrm`, `prebuild` | Path to the temporary directory. This directory is cleared automatically when the script is finished.
`VARFILE` | `prerm`, `postrm` | Path to the file containing the values of variables saved using the `setvar` subcommand (run `debcomp setvar --help` command for details). If this file does not exist, the variable will not be declared.
`CONF_<PARAM>` | `preinst`, `postinst`, `prerm`, `postrm` | Value of the component configuration parameter (see &laquo;[Component configuration](#component-configuration)&raquo;). The `<PARAM>` part is the name of the configuration parameter in uppercase.


## Component configuration

Keys that are only valid for the "common" component":

Key | Description
--- | ---
`osrelease` | If it exists, the name and version of the operating system specified in the variables `ID` and `VERSION_ID` in the file `/etc/os-release`, or `/usr/lib/os-release`, if the first one does not exist, will be checked for compliance with the specified value. Example: `osrelease=ubuntu 20.04`.

Keys that are valid for all components except "common":

Key | Description
--- | ---
<code>depends/&lt;component&gt;&#124;&lt;component&gt;&#124;...</code> | Declares one of the components (`<component>`) that the current component depends on, or a group of components separated by <code>&#124;</code> character. If a group is specified, at least one of the listed components must be installed to satisfy the dependency. The key does not require a value to be specified.
`conflicts/<component>` | Declares one of the components (`<component>`) that the current component conflicts with. The key does not require a value to be specified.
`after/<component>` | Declares one of the components (`<component>`) that must be installed before the current component. The key does not require a value to be specified.

Keys that are valid for all components:

Key | Description
--- | ---
`arch/<dpkg_arch>` | If `arch` exists, the operating system architecture will be checked (the result of executing the `dpkg --print-architecture` command) for compliance with any of the listed architectures. The `<dpkg_arch>` part corresponds to one of the valid architectures. The key does not require a value to be specified.
`name[<lang>]` | Display name of the component. The optional `[<lang>]` part is used to specify a localized variant of the value.
`description[<lang>]` | Description of the component. The optional `[<lang>]` part is used to specify a localized variant of the value.
`conf/<param>/name[<lang>]` | Name of the configuration parameter. The optional `[<lang>]` part is used to specify a localized variant of the value.
`conf/<param>/description[<lang>]` | Description of the configuration parameter. The optional `[<lang>]` part is used to specify a localized variant of the value.
`conf/<param>/type` | Type of the configuration parameter value. It can take the following values: `string`, `integer`, `float`, `boolean`, `select`.
`conf/<param>/default` | Default value of the configuration parameter.
`conf/<param>/hidden` | If specified and set to 1, the value of the configuration parameter will not be displayed in the user interface and error messages. Applies only to `string`, `integer` and `float` types.
`conf/<param>/regexp` | A regular expression that must match the value of the configuration parameter, otherwise the configuration is considered invalid. Applies only to `string`, `integer` and `float` types.
`conf/<param>/min` | If specified, the value of the configuration parameter must not be less than the specified value, otherwise the configuration is considered invalid. Applies only to `integer` and `float` types.
`conf/<param>/max` | If specified, the value of the configuration parameter must not be greater than the specified value, otherwise the configuration is considered invalid. Applies only to `integer` and `float` types.
`conf/<param>/options/<value>` | If the value type of the configuration parameter is `select`, it declares one of the possible values (`<value>`) of the configuration parameter. The key does not require a value to be specified.
`conf/<param>/options/<value>/name[<lang>]` | Display name of the configuration parameter value. The optional `[<lang>]` part is used to specify a localized variant of the value.
`conf/<param>/options/<value>/description[<lang>]` | Description of the value of the configuration parameter. The optional `[<lang>]` part is used to specify a localized variant of the value.

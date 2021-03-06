How to refresh avocado module for new upstream release
======================================================

This is an outline of the steps to update the Fedora/EPEL ``avocado:latest``
module stream when there is a new upstream release of ``avocado``.
This example is based on updating from 74.0 to 75.1.

Update downstream python-avocado package
----------------------------------------

#. Use pagure to create a personal fork of the downstream Fedora dist-git
   ``python-avocado`` package source repository
   https://src.fedoraproject.org/rpms/python-avocado
   if you don’t already have one.

#. Clone your personal fork repository to your local workspace.

#. Checkout the ``latest`` branch--which is the stream branch used by the
   ``avocado:latest`` module definition.
   Make sure your ``latest`` branch is in sync with the most recent commits
   from the official dist-git repo you forked from.

#. Locate the official upstream commit hash and date corresponding to the
   upstream GitHub release tag.
   (eg., https://github.com/avocado-framework/avocado/releases/tag/75.1)
   Use those values to update the ``%global commit`` and ``%global commit_date``
   lines in the downstream ``python-avocado.spec`` file.

#. Update the ``Version:`` line with the new release tag.

#. Reset the ``Release:`` line to ``1%{?gitrel}%{?dist}``.

#. Add a new entry at the beginning of the ``%changelog`` section with a message
   similar to ``Sync with upstream release 75.1.``.

#. See what changed in the upstream SPEC file since the last release.
   You can do this by comparing branches on GitHub
   (eg., https://github.com/avocado-framework/avocado/compare/74.0..75.1)
   and searching for ``python-avocado.spec``.
   If there are changes beyond just the
   ``%global commit``, ``%global commit_date``, and ``Version:`` lines,
   and the ``%changelog`` section,
   make any necessary corresponding changes to the downstream SPEC file.
   Note: the commit hash in the upstream SPEC file *will* be different that
   what gets put in the downstream SPEC file since the upstream hash was added
   to the file before the released commit was made.
   Add an additional note to your ``%changelog`` message if there were any
   noteworthy changes.

#. Download the new upstream source tarball based on the updated SPEC by
   running::

    spectool -g python-avocado.spec

#. Add the new source tarball to the dist-git lookaside cache and update your
   local repo by running::

    fedpkg new-sources avocado-75.1.tar.gz

#. Create a Fedora source RPM from the updated SPEC file and tarball by running::

    fedpkg --release f31 srpm

   This example is starting with the latest supported Fedora release (f31).
   Expect several ``warning: extra tokens at the end of %endif directive in line``
   messages until
   https://github.com/rpm-software-management/rpm/issues/829 gets fixed.
   It should write an SRPM file (eg., ``python-avocado-75.1-1.fc31.src.rpm``)
   to the current directory.

#. Test build the revised package locally using ``mock``.
   This gets a little tricky because the package dependency ``python-aexpect``
   is part of the ``avocado`` module--which isn’t automatically available in
   the mock build environment.
   Run the build using the same Fedora release for which the SRPM was created::

    mock -r fedora-31-x86_64 --init
    mock -r fedora-31-x86_64 --enablerepo=fedora-modular --dnf-cmd module enable avocado:latest
    mock -r fedora-31-x86_64 --enablerepo=fedora-modular --dnf-cmd install python3-aexpect
    mock -r fedora-31-x86_64 --enablerepo=fedora-modular --dnf-cmd module disable avocado
    mock -r fedora-31-x86_64 --no-clean python-avocado-75.1-1.fc31.src.rpm

#. If the package build fails, go back and fix the SPEC file, re-create the SRPM,
   and retry the mock build.
   It is occasionally necessary to create a patch to disable specific tests
   or pull in some patches from upstream to get the package to build correctly.
   See https://src.fedoraproject.org/rpms/python-avocado/tree/52lts as an example.

#. Repeat the SRPM generation and mock build for all other supported Fedora
   releases, Fedora Rawhide, and EPEL8.
   For Rawhide enable mock repo ``rawhide-modular`` instead.
   For EPEL8 use mock release ``epel-8-x86_64`` and
   enable repo ``epel-modular`` instead.
   Note: enabling the mock ``epel-modular`` repo for EPEL8 will not be possible
   until https://github.com/rpm-software-management/mock/pull/455
   is released--unless you manually copy the file modified by that PR
   into place on your system.
   If you’re feeling confident, you can skip this whole step.

#. When you have successful builds for all releases,
   ``git add``, ``git commit``, and ``git push`` your updates.


Update downstream avocado module
--------------------------------

#. Use pagure to create a personal fork of the downstream Fedora dist-git
   ``avocado`` module source repository
   https://src.fedoraproject.org/modules/avocado
   if you don’t already have one.

#. Clone your personal fork repository to your local workspace.

#. Checkout the ``latest`` branch--which the stream branch used for the
   ``avocado:latest`` module definition.
   Make sure your ``latest`` branch is in sync with the latest commits to
   the official dist-git repo you forked from.

#. If there are any new ``python-avocado`` sub-packages,
   adjust the ``avocado.yaml`` modulemd file accordingly.

#. Test with a scratch module build for the latest supported
   Fedora release (f31),
   including the SRPM created earlier::

    fedpkg module-scratch-build --requires platform:f31 --buildrequires platform:f31 --file avocado.yaml --srpm .../python-avocado/python-avocado-75.1-1.fc31.src.rpm

   You can use https://release-engineering.github.io/mbs-ui/ to monitor the
   build progress.

#. If the module build fails, go back and fix the modulemd file and try again.
   Depending on the error, it may necessary to go back and revise the package
   SPEC file.

#. Repeat the scratch module build for all other supported Fedora releases,
   Fedora Rawhide, and EPEL8 (``platform:el8``).
   If you’re feeling confident, you can skip this step.

#. When you have successful scratch module builds for all releases,
   ``git add``, ``git commit``, ``git push`` your update.
   Note: if ``avocado.yaml`` didn’t need modifying, it is still necessary to
   make a new commit since official module builds are tracked internally by
   their git commit hash.
   Recall that ``git commit`` has an ``--allow-empty`` option.

Release revised module
----------------------

#. Create PRs to merge the ``python-avocado`` rpm and ``avocado`` module changes
   into the ``latest`` branches of the master dist-git repositories.
   If you have commit privileges to the master repositories, you could also opt
   to push directly.

#. After the ``python-avocado`` rpm and ``avocado`` module changes have been merged...

#. From the ``latest`` branch of your module repository in your local workspace,
   submit the module build using ``fedpkg module-build``.
   The MBS (Module Build Service) will use stream expansion to automatically
   build the module for all current Fedora/EPEL releases.
   Again, you can use https://release-engineering.github.io/mbs-ui/
   to monitor the progress of the builds.

#. If you want to test the built modules at this point, use ``odcs``
   (On Demand Compose Service) to create a temporary compose for your
   Fedora release::

    odcs create module avocado:latest:3120200121201503:f636be4b

   You can then use ``wget`` to download the repofile from the URL referenced
   in the output to ``/etc/yum.repos.d/`` and then you’ll be able to install
   your newly built ``avocado:latest`` module.
   Don't forget to remove the odcs repofile when you are done testing.

#. Use https://bodhi.fedoraproject.org/ to create new updates for
   ``avocado:latest`` (using options type=enhancement, severity=low,
   default for everything else) for each Fedora release and EPEL8--except
   Rawhide which happens automatically.

#. Bodhi will push the updates to the testing repositories in a day or two.
   Following the push and after the Fedora mirrors have had a chance
   to sync, you'll be able to install the new module by including the
   ``dnf`` option ``--enablerepo=updates-testing-modular``
   (``epel-testing-modular`` for EPEL).

#. After receiving enough bodhi karma votes (three by default) or after
   enough days have elapsed (seven for Fedora, twelve for EPEL), bodhi
   will push the updated modules to the stable repositories.
   At that point, the updated modules will be available by default without any
   extra arguments to ``dnf``.

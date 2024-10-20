#######################
HOWTO of systemd-repart
#######################

|

Overview
########

This demonstrates using systemd-repart_ to dynamically create partitions.  This approach can be
used when generating a minimized system image for tiny size and portability.  The system image can
then be written to a larger storage device and grown to fill space as well as create new partitions
that do not need to take up space in the original image (such as a swap partition or data
partition).

|

Files
#####

::

    repart_demo
      |
      +- README.rst     (this file)
      +- build-clear/   (temporary mkosi dir for building the "clear" demo)
      +- build-crypt/   (temporary mkosi dir for building the "crypt" demo)
      +- cache/         (temporary dir that caches mkosi files)
      +- key.bin        (generated encryption key for demonstration purposes)
      +- mkosi-base/    (base files used by both demonstrations)
      +- mkosi-crypt/   (additional files used by the "crypt" flavor)
      +- demo           (script to run the demonstration)

|

Quick Start
###########

There are two example "flavors":

:clear: Demonstrate first-boot partitioning sans encryption
:crypt: Demonstrate first-boot partitioning with encryption

Each of these examples is built and run using the ``demo`` tool (see `demo Tool Dependencies`_) in the
top directory with the flavor as an argument::

    >$ ./demo clear

The ``demo`` tool progresses as follows:

1. setup a mkosi_ configuration in a build directory ``build-<FLAVOR>`` - in the above case
   ``build-clear``
2. build a disk image  (``build-<FLAVOR>/image.raw``) by executing ``mkosi`` on the configuration
3. expand the disk image as if it were written to a larger storage device [IMAGEEXP]_
4. run the image in a virtual machine using qemu_
5. shut down the virtual machine after 30 seconds of up-time

At various points along the way the image partition details will be emitted using disktype_.

.. NOTE:: It is not intended for the virtual machines to be long-running processes for the user to
          explore.  It is intended that the resulting ``image.raw`` final state after all of the
          listed steps execute are what can be examined and reviewed.

|

Mechanics
#########

Working system images are generated with mkosi_ and use Ubuntu 24.10 [UBUNTU24]_.
These are minimal images to avoid cluttering the details of the demonstration.  The generated images
are then resized as if they were written to a large device and then "booted" using QEMU_ and KVM_
virtualization.  While all of this is driven using the ``demo`` tool from the top directory the
mechanisms and results are best observed by inspecting the output and the resulting ``image.raw``
artifact.  Additionally pulling-apart the ``image.raw`` will allow ``systemd`` journal events and
messages to be reviewed.

|

.. _General Partitioning:

General Partitioning
====================

The initial disk image is generated using partitions described in ``mkosi-base/mkosi.repart``.  It
is to be composed of an ``ESP`` (EFI System Partition) and a rootfs that are minimal in size.

The general partitioning approach can be described as *late* partitioning.  It runs once the target
root file system (rootfs) is mounted and services are executing from the rootfs.  An *early*
partitioning approach would be one that is performed while the running in the initial RAM disk
(``initrd``) prior switching to and executing from the target rootfs.

Due to this late approach the partitioning must be performed prior to full system activity - this
should be *before* the ``systemd`` ``basic.target``.  This is driver from the
``systemd-repart.service``.  If partitions are created at this time the new partitions will need to
be mounted.  For simplicity this example chooses to simply reboot the system with the new partitions
so that all start-up mechanisms and services will function with the new partitions.  Subsequent
start-ups will not create new partitions (they were all created the first time) and will continue to
the ``systemd`` ``default.target``.

Shortly after boot and services are executing from the target rootfs ``systemd-repart.service`` is
invoked.  The following files (which originate in ``mkosi-base/mkosi.extra``) influence how the
service works:

* ``/usr/lib/repart.d``
* ``/usr/lib/systemd/system/systemd-repart.service.d/systemd-repart-gen_tabs.conf``
* ``/usr/lib/systemd/system/update-tabs.service``
* ``/usr/lib/systemd/system/test-poweroff.service``

|

repart.d
++++++++

This contains configuration (``.conf``) files that describe expected partitions - one partition per
``.conf`` file.  The configuration structure is describe in man page ``repart.d(5)``.  Expanding an
existing partition can be done by setting ``SizeMaxBytes=`` in conjunction with ``Priority=`` and
``Weight=``.  Adding a new partition can be performed by listing the new partition in an additional
``.conf`` file.

.. NOTE:: ``.conf`` files are read in lexicographical-sorted order from the ``repart.d`` directory.

.. WARNING:: There must be a matching ``.conf`` file in ``repart.d`` for each *existing* partition -
             otherwise new partitions will not be created.

The ``30-xdata.conf`` file in ``repart.d`` creates a new partition that is mounted at ``/xdata``.

|

systemd-repart.service.d/systemd-repart-gen_tabs.conf
+++++++++++++++++++++++++++++++++++++++++++++++++++++

The ``systemd-repart.service.d/systemd-repart-gen_tabs.conf`` file overrides the typical way that
``systemd-repart`` is invoked and adds arguments of ``--generate-fstab=/run/fstab``.  This
"captures" what ``systemd-repart`` expects for entries in ``/etc/fstab``.  It also ensures that
the ``/run`` mount is available for recording the generated ``fstab``.  All other details for running
``systemd-repart.service`` are inherited from the vendor-supplied file in
``/usr/lib/systemd/system/systemd-repart.service``.

|

update-tabs.service
+++++++++++++++++++

This service is started after ``systemd-repart.service`` has completed.  It is responsible for
comparing the newly-generated ``fstab`` in ``/run/fstab`` against the previous rootfs
``/etc/fstab``.  If the ``fstab`` has changed (*viz* new partitions have been created) then the
generated ``fstab`` is copied to ``/etc/fstab`` and a reboot is triggered so that all partitions
will be mounted for the subsequent boot.

|

/usr/lib/systemd/system/test-poweroff.service
+++++++++++++++++++++++++++++++++++++++++++++

This is a ``systemd`` service that powers of the system after 30 seconds of up-time.

|

Encryption
==========

Creating encrypted partitions proceeds in a similar way as the `General Partitioning`_ described
above.  There are some differences with files that are key to the ``systemd-repart`` process: some
are different versions of what was used in General Partitioning and some are additional files.
These are the specifics:

* ``/usr/lib/repart.d``
* ``/usr/lib/systemd/system/systemd-repart.service.d/systemd-repart-gen_tabs.conf``
* ``/usr/lib/systemd/system/update-tabs.service``
* ``/usr/lib/systemd/system/test-poweroff.service``
* ``/usr/lib/systemd/system/fscrypt.socket``
* ``/usr/lib/systemd/system/fscrypt@.service``
* ``/usr/bin/getkey``
* ``/var/lib/key.bin``

|

repart.d
++++++++

The ``30-xdata.conf`` file in ``repart.d`` is nearly identical to the same file used in the "clear"
flavor.  The difference is the addition of the following lines::

  Encrypt=key-file
  EncryptedVolume=dc-xdata:/run/fscrypt.sock:luks,discard

The ``Encrypt=key-file`` indicates that the encryption key will come from the output of an
executable.  The output is obtained from the socket ``/run/fscrypt.sock`` specified in the
``EncryptedVolume=`` value.  More about how that mechanism works is described below by
`fscrypt.socket and fscrypt@.service`_.

|

systemd-repart.service.d/systemd-repart-gen_tabs.conf
+++++++++++++++++++++++++++++++++++++++++++++++++++++

This is similar to the "clear" case above except it also adds `` --generate-crypttab=/run/crypttab``
as an option for generating a new ``crypttab``.

|

update-tabs.service
+++++++++++++++++++

This is similar to the "clear" case above except it also compares ``crypttab`` in the same way as
``fstab``.

|

/usr/lib/systemd/system/test-poweroff.service
+++++++++++++++++++++++++++++++++++++++++++++

This works the same way as the "clear" flavor and powers-off the system after 30 seconds.

|

.. fscrypt.socket and fscrypt@.service:

fscrypt.socket and fscrypt@.service
+++++++++++++++++++++++++++++++++++

To utilize an executable to obtain an encryption key requires coordinated units in ``systemd``.  One
is a ``.socket`` unit and the other is a ``.service`` unit of the same name.  The ``.socket`` unit
sets up a socket end-point - in this case a UNIX-domain socket at ``/run/fscrypt.sock``.  When a
process reads from that socket the ``.service`` of the same name is started to produce the output -
in this case ``/usr/bin/getkey``.

|

getkey and key.bin
++++++++++++++++++

.. WARNING:: The obtaining of the encryption key using ``getkey`` and ``key.bin`` is not intended to
             e a pattern for strong encryption - in fact it is trivially discovered and stolen in
             this case.  This mechanism is for demonstration purposes only.  An implementer is
             responsible for creating an appropriate mechanism for secure encryption.  One possible
             method would be to use a TPM [TPM]_.

The ``getkey`` executable demonstrates how a key can be obtained by calling a process which produces
the key to ``stdout``.  In this case it simply cats ``key.bin`` to ``stdout``.

The ``key.bin`` file is automatically generated from ``/dev/random`` and folded-into the "crypt"
demonstration by ``demo``.

|

Additional Topics
#################

* Growing File Systems
* Populating New File Systems
* Factory Reset File Systems
* Read-only RootFS
* A-B Booting
* TPM Secrets

|

Appendices
##########

|

.. _demo Tool Dependencies:

demo Tool Dependencies
======================

The ``demo`` tool, while simple, has a few dependencies that need to be installed and configured:

* mkosi_: make bespoke operating system images
* QEMU_: full system emulation
* KVM_: Linux virtualization
* disktype_: detection of content format of a disk or disk image

These are usually available as distribution vendor packages and can often be trivially installed.

The ``mkosi`` version must be at least 24 and is enforced by ``mkosi-base/mkosi.conf``.

|

Debian and Ubuntu
+++++++++++++++++

The following commands are *adminstrative* commands and will need proper authorization escalation
(possibly using ``sudo``) to execute them.

::

  >$ apt install mkosi disktype qemu-system-x86

The user account running the examples will need to be added to the ``kvm`` group::

  >$ adduser <USERNAME> kvm


..
  Endnotes

.. [IMAGEEXP] The ``image.raw`` generated by ``mkosi`` is expanded to a larger size to
                         simulate it being written to a storage device that is significantly larger
                         than the original ``image.raw``.  This is done using the ``dd`` utility to
                         open the file and seek beyond the end of the file to the desired size and
                         write a single chunk (*viz* block) of zeros (from ``/dev/zero``).  This
                         creates what is known as a "sprse file" which appears to be larger than the
                         actual blocks of data recorded on the storage device since the unwritten
                         blocks do not exist.  The unwritten blocks (or holes) can be read and will
                         be implicit blocks of zero.

.. [UBUNTU24] The Ubuntu 24.10 distribution was selected as the base because it has a resent
   ``systemd`` version.  A minimum version of ``systemd`` 256 is necessary for various features of
   ``systemd-gpt-auto-generator`` (``systemd.swap=``, ``systemd.image_policy=`` added in 254) and
   necessary features in ``systemd-repart`` (``EncryptedVolume=`` and ``MountPoint=`` added in 256).
   It is assumed that any distribution that meets this requirement can work with a
   reasonably-adapted implementation of what is described in this document.


.. [TPM] ``systemd-cryptsetup`` and related tooling support setting up a TPM and making direct-use
         of it.

|

References
##########


..
  Links

.. _systemd-repart: https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.html
.. _mkosi: https://github.com/systemd/mkosi
.. _QEMU: https://www.qemu.org/
.. _KVM: https://linux-kvm.org/
.. _disktype: https://disktype.sourceforge.net/


..
   Local Variables:
   fill-column: 100
   End:

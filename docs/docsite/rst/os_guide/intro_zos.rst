:orphan:

Managing z/OS hosts with Ansible
================================


Ansible can connect to UNIX Systems Services to bring your Ansible Automation strategy to IBM Z.

"This enables development and operations automation on Z through a seamless, unified workflow orchestration with configuration management, provisioning, and application deployment in one easy-to-use platform."


Ansible and UNIX Systems Services
---------------------------------
USS is POSIX compliant, so it can support Ansible dependencies like making SSH connections, spawning shell processes, and python. 
With these things, Ansible can be run on USS to modify files, directories, etc. 
Anything that one could do by SSH-ing into USS and typing can be captured and automated in an Ansible playbook.

For additional functionality with z/OS managed nodes, check out the `Red Hat Certified Content for IBM Z <https://ibm.github.io/z_ansible_collections_doc/>`_.
To learn more about z/OS managed nodes, see `Red Hat Certified Content for IBM Z <https://ibm.github.io/z_ansible_collections_doc/>`_.


Dealing with EBCDIC
-------------------
USS files largely come in three flavors - binary, utf-8 encoded text, and ebcdic-encoded text.
Ansible has provisions to handle binary and UTF-8, but not EBCDIC. 
It is up to the Ansible user managing z/OS nodes to understand which files may be affected.
This is not necessarily a limitation, it simply requires additional steps in defining additional tasks convert files to/from their original encodings.


File Tags
---------

This is a z/OS Unix Systems Services concept (part of enhanced ASCII) established to distinguish binary files from utf-8 encoded text files and ebcdic-encoded text files.

The type (binary or text) and encoding of the data can be stored in a tag (a feature of enhanced ASCII). 
Default behavior for an un-tagged file or stream is determined by the program, for example, 
`IBM Open Enterprise SDK for Python <https://www.ibm.com/products/open-enterprise-python-zos>`__ defaults to the UTF-8 encoding.

Ansible modules will not do this automatically, but you do it yourself with an additional task using the builtin.command module.

.. code-block::
    - name: tag my_file.txt as ibm-1047 ebcdic.
      ansible.builtin.command: chtag -tc ibm1047 my_file.txt

idk
---

This means that data sent to remote z/OS nodes is encoded in UTF-8 and is not tagged.
The z/OS UNIX remote shell defaults to an EBCDIC encoding for un-tagged data streams. 
This mismatch in data encodings can be resolved with the ``PYTHONSTDINENCODING`` environment variable,
which tags the pipe with the encoding specified. 
File and pipe tags are used for automatic conversion between ASCII and EBCDIC.



Using Ansible Community Modules with z/OS
-----------------------------------------

The ansible core engine is unaware of tags in z/OS UNIX and thus processes all data as either binary or text encoded in UTF-8.
The Ansible community modules assume all data (files and pipes/streams) is utf-8 encoded (if not binary).

On z/OS, data (file or stream) is found in one of three ways: binary, as text encoded in UTF-8, or as text encoded in EBCDIC.

Here are some notes / pro-tips when using the community modules with z/OS. This is by no means a comprehensive list.

* ansible.builtin.command / ansible.builtin.shell

* ansible.builtin.raw

    The raw module, by design, ignores all remote environment settings.
    One trick to pass in the bare minimal environment variables is to chain export statements before the desired command. 

    .. code-block:: yaml

        ansible.builtin.raw: |
            export _BPXK_AUTOCVT: "ON" ;
            export _CEE_RUNOPTS: "FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)" ;
            export _TAG_REDIR_ERR: "txt" ;
            export _TAG_REDIR_IN: "txt" ;
            export _TAG_REDIR_OUT: "txt" ;
            echo "hello world!"

    Alternatively, consider using the ansible.builtin.command or ansible.builtin.shell modules mentioned above.

* ansible.builtin.copy / ansible.builtin.fetch
    The built in community modules will NOT automatically tag files, nor will existing file tags be honored nor preserved.

* ansible.builtin.blockinfile / ansible.builtin.lineinfile
    These modules process all data in UTF-8, so be sure to convert files before and re-tag the resulting files after.

* ansible.builtin.replace - ketan doesn't know (yet)

* ansible.builtin.script - won't work - file tagging issue.



Configuring the Remote Environment
-----------------------------------

Certain Language Environment (LE) configurations enable automatic encoding conversion and automatic file tagging functionality required by python on z/OS systems.

Include the following configurations when setting the remote environment for any z/OS managed nodes. (group_vars, host_vars, playbook, or task):

.. code-block:: yaml

    _BPXK_AUTOCVT: "ON"
    _CEE_RUNOPTS: "FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)"

    _TAG_REDIR_ERR: "txt"
    _TAG_REDIR_IN: "txt"
    _TAG_REDIR_OUT: "txt"


Note, the remote environment can be set any of these levels:
* inventory - inventory.yml, group_vars/all.yml, or host_vars/all.yml
* playbook - ``environment`` variable at top of playbook.
* block or task - ``environment`` key word.

See <here> for more details on setting environment variables. TODO - link to ansible docs on environment config.

Configuring the Remote Python Interpreter
-----------------------------------------

Ansible requires a python interpreter to run most modules on the remote host, and it checks for python at the ‘default’ path ``/usr/bin/python``.

On z/OS, the python3 interpreter (from `IBM Open Enterprise SDK for Python <https://www.ibm.com/products/open-enterprise-python-zos>`_) is often installed to a different path, typically something like: 
``<path-to-python>/usr/lpp/cyp/v3r12/pyz``.

This path to the python interpreter can be configured with the Ansible inventory variable ``ansible_python_interpreter``.
For example:

.. code-block:: ini

    zos1 ansible_python_interpreter:/python/3.12/usr/lpp/cyp/v3r12/pyz

When the path to the python interpreter is not found in the default location on the target host, an error containing the following message may result: ``/usr/bin/python: FSUM7351 not found``

For more details, see: :ref:`python_interpreters`. TODO - link should be to FAQ page (not loca)

Enabling Ansible Pipelining
---------------------------
Enable pipelining in the ansible.cfg file. TODO - <link to pipelining config>

When Ansible pipelining is enabled (`see the config option here <https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-pipelining>`_),
Ansible passes any module code to the remote target node through python's stdin pipe and runs it in all in a single call rather than copying data to temp files and reading from those files.
For more details on pipelining, see: :ref:`flow_pipelining`.

Enabling this behavior is encouraged because python will tag its pipes with the proper encoding, so there is less chance of encountering encoding errors. 
Further, using python stdin pipes is more performant than file I/O.


Include the following in the environment for any tasks performed on z/OS target nodes.
The value should be the encoding used by the z/OS UNIX shell on the remote target.

.. code-block:: yaml

    PYTHONSTDINENCODING: "cp1047"

When Ansible pipelining is enabled but the ``PYTHONSTDINENCODING`` property is not correctly set, the following error may result.
Note, the ``'\x81'`` below may vary based on the target user and host:

.. code-block::
    SyntaxError: Non-UTF-8 code starting with '\\x81' in file <stdin> on line 1, but no encoding declared; see https://peps.python.org/pep-0263/ for details


idk-Dealing with unreadable chars
-----------------------------

You're probably running into an EBCDIC encoding mix up.
Double check that your remote environment is set up properly.
Also check the expected file encodings, both on the remote node and the controller.
ansible-core modules will assume all text data is utf8 encoded, while z/OS may be using EBCDIC.
On many z/OS systems, the default encoding for untagged files is EBCDIC.
This variation in default settings can easily lead to interpreting data using the the wrong encoding.

:orphan:

Managing z/OS hosts with Ansible
================================


Ansible can connect to UNIX Systems Services to bring your Ansible Automation strategy to IBM Z.

"This enables development and operations automation on Z through a seamless, unified workflow orchestration with configuration management, provisioning, and application deployment in one easy-to-use platform."


Ansible and UNIX Systems Services
---------------------------------
UNIX Systems Services can support the required dependencies for an Ansible managed node including running python and spawning interactive shell processes through an SSH connection.
Ansible can target UNIX Systems Services nodes to modify files, directories, etc. through built-in Ansible community modules. Further, 
anything that one can do by typing command(s) into shell can be captured and automated in an Ansible playbook.

To learn more about z/OS managed nodes, see `Red Hat Certified Content for IBM Z <https://ibm.github.io/z_ansible_collections_doc/>`_.


The z/OS Landscape
-------------------
While most systems process files in two modes - binary or utf8 encoded text, IBM Z and UNIX Systems Services features an additional third flavor - ebcdic-encoded text.
Ansible has provisions to handle binary data and UTF-8 encoded textual data, but not data encoded in EBCDIC.
This is not necessarily a limitation, it simply requires additional steps in defining additional tasks convert files to/from their original encodings.
It is up to the Ansible user managing z/OS nodes to understand the nature of the files in their automation.

The type (binary or text) and encoding of files can be stored in "tags". File tags is a z/OS UNIX Systems Services concept (part of enhanced ASCII) which was established to distinguish binary files from utf-8 encoded text files and ebcdic-encoded text files.

Default behavior for an un-tagged file or stream is determined by the program, for example, 
`IBM Open Enterprise SDK for Python <https://www.ibm.com/products/open-enterprise-python-zos>`__ defaults to the UTF-8 encoding.

Ansible modules will not read or honor any file tags. It is up to the user to determine the nature of remote data. This is achieveable with an additional task using the ``builtin.command`` module and apply any necessary encoding conversion.

.. code-block:: yaml

    - name: tag my_file.txt as ibm-1047 ebcdic.
      ansible.builtin.command: chtag -tc ibm1047 my_file.txt


Data sent to remote z/OS nodes is by default encoded in UTF-8 and is not tagged.
The z/OS UNIX remote shell defaults to an EBCDIC encoding for un-tagged data streams. 
This mismatch in data encodings can be resolved with the ``PYTHONSTDINENCODING`` environment variable,
which tags the pipe with the encoding specified. 
File and pipe tags can be used for automatic conversion between ASCII and EBCDIC. But only by programs which are aware of tags and honor them.


Using Ansible Community Modules with z/OS
-----------------------------------------

The Ansible community modules assume all textual data (files and pipes/streams) is utf-8 encoded.

On z/OS, since text data (file or stream) is sometimes encoded in EBCDIC and sometimes in UTF-8, special care must be taken to identify the encoding of target data.

Here are some notes / pro-tips when using the community modules with z/OS. This is by no means a comprehensive list.

* ansible.builtin.command / ansible.builtin.shell
    The command and shell modules are excellent for automating tasks for which command line solutions already exist. 
    The thing to keep in mind when using these modules is depending on the system configuration, the Z shell (/bin/sh) may return output in EBCDIC.
    The LE environment variable configurations will correctly convert streams if they are tagged and make output look sensible on the Ansible side.
    However, some command line programs may return output in UTF-8 and not tag the pipe.
    In this case, the autoconversion may incorrectly assume output is in EBCDIC and attempt to convert it and wind up with yet more unreadable data.
    If the source encoding can be determined, since the ``ansible.builtin.command`` module allows chaining together commands through pipes, try piping the output through a call to iconv.
    You may need to play around with the 'to' and 'from' encodings to determine the correct set.

    .. code-block:: yaml

        ansible.builtin.command: "some_pgm | iconv -f ibm-1047 -t iso8859-1"


* ansible.builtin.raw
    The raw module, by design, ignores all remote environment settings. Running against UNIX Systems Services as a managed nodes requires some base configurations.
    One trick to use this module with UNIX Systems Services is to pass in the bare minimal environment variables as a chain of export statements before the desired command.

    .. code-block:: yaml

        ansible.builtin.raw: |
            export _BPXK_AUTOCVT: "ON" ;
            export _CEE_RUNOPTS: "FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)" ;
            export _TAG_REDIR_ERR: "txt" ;
            export _TAG_REDIR_IN: "txt" ;
            export _TAG_REDIR_OUT: "txt" ;
            echo "hello world!"

    Alternatively, consider using the ``ansible.builtin.command`` or ``ansible.builtin.shell`` modules mentioned above,
    which set up the configured environment for each task.


* ansible.builtin.copy / ansible.builtin.fetch
    The built in community modules will NOT automatically tag files, nor will existing file tags be honored nor preserved.
    You can treat files as binaries when running copy/fetch operations, there is no issue in terms of data integrity,
    just remember to restore the correct tag and encoding once the file is returned to z/OS, as that data will not be stored for you.

* ansible.builtin.blockinfile / ansible.builtin.lineinfile
    These modules process all data in UTF-8, so be sure to convert files before and re-tag the resulting files after.

* ansible.builtin.script
    The built in script module copies a local file over to a remote target and attempts to run it.
    The issue that UNIX Systems Services targets run into is that the file does not get tagged as UTF-8 text.
    When the underlying shell attempts to read the untagged script file, it will assume the default,
    that the file is encoded in EBCDIC, and the file will not be read correctly and the script will not run.
    One work-around is to manually copy local files over (``ansible.builtin.copy`` ) and convert or tag files (with the ``ansible.builtin.command`` module).
    With this work-around, some of the niceties of the script module are lost, such as automatically cleaning up the script file once it's run,
    but it is trivial to recreate those steps as separate playbook tasks.

    .. code-block:: yaml

        - name: Copy local script file to remote
            ansible.builtin.copy:
                src: "{{ playbook_dir }}/local/scripts/sample.sh"
                dest: /u/ibmuser/scripts/

        - name: Tag remote script file
            ansible.builtin.command: "chtag -tc ISO8859-1 /u/ibmuser/scripts/sample.sh"

        - name: Run script
            ansible.builtin.command: "/u/ibmuser/scripts/sample.sh"

    Another convoluted work-around is to store local script files in EBCDIC.
    They may be unreadable on the controller, but they will copy over to UNIX Systems Services targets,
    be read in correctly as EBCDIC, and the script will run. This approach takes advantage of the built-in conveniences of the script module,
    but storing unreadable files locally makes maintaining those script files difficult.

Configure the Remote Environment
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

For more details, see :ref:`playbooks_environment`.

Configure the Remote Python Interpreter
----------------------------------------

Ansible requires a python interpreter to run most modules on the remote host, and it checks for python at the ‘default’ path ``/usr/bin/python``.

On z/OS, the python3 interpreter (from `IBM Open Enterprise SDK for Python <https://www.ibm.com/products/open-enterprise-python-zos>`_) is often installed to a different path, typically something like: 
``<path-to-python>/usr/lpp/cyp/v3r12/pyz``.

This path to the python interpreter can be configured with the Ansible inventory variable ``ansible_python_interpreter``.
For example:

.. code-block:: ini

    zos1 ansible_python_interpreter:/python/3.12/usr/lpp/cyp/v3r12/pyz

When the path to the python interpreter is not found in the default location on the target host, an error containing the following message may result: ``/usr/bin/python: FSUM7351 not found``

For more details, see: :ref:`python_interpreters`.

Enable Ansible Pipelining
---------------------------
Enable :ref:`ANSIBLE_PIPELINING` in the ansible.cfg file.

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
---------------------------------

You're probably running into an EBCDIC encoding mix up.
Double check that your remote environment is set up properly.
Also check the expected file encodings, both on the remote node and the controller.
ansible-core modules will assume all text data is utf8 encoded, while z/OS may be using EBCDIC.
On many z/OS systems, the default encoding for untagged files is EBCDIC.
This variation in default settings can easily lead to interpreting data using the the wrong encoding.

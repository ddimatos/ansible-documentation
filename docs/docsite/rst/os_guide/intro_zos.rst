:orphan:

Managing z/OS hosts with Ansible
================================

Hi! Welcome to the z/OS page.
This page is under construction ⚠️.

USS not MVS
-----------

Out of the box, Ansible can connect to and drive automation against z/OS UNIX Systems Services (USS).

For additional functionality with z/OS managed nodes, check out the Red Hat Certified Content for IBM Z.


Setting up the remote environment for a z/OS host
-------------------------------------------------

* Certain language environment (LE) configurations enable auto conversion and file tagging functionality required by python on z/OS systems. 

  Include the following configurations when setting the remote environment for any z/OS managed nodes. (group_vars, host_vars, playbook, or task):

  .. code-block:: yaml

    _BPXK_AUTOCVT: "ON"
    _CEE_RUNOPTS: "FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)"

    _TAG_REDIR_ERR: "txt"
    _TAG_REDIR_IN: "txt"
    _TAG_REDIR_OUT: "txt"


  Note, the remote environment can be set any of these levels: inventory (inventory.yml, group_vars, or host_vars), play, block, or task with the ``environment`` key word.


Known Work Arounds
------------------
The Ansible community modules assume all data is utf-8 encoded (if not binary). 

* ansible.builtin.command, ansible.builtin.shell
* ansible.builtin.raw
    The raw module 
* ansible.builtin.copy, ansible.builtin.fetch
    The built in community modules will NOT automatically tag files, nor will existing file tags be honored nor preserved.

How does Ansible handle EBCDIC?
-------------------------------

It doesn't. But magical LE (language environment) configurations can help get around this.

* IBM z/OS processes data (file or instream) primarily in one of three ways: binary, as text encoded in UTF-8, or as text encoded in EBCDIC. The type (binary or text) and encoding of the data can be stored in a tag (a feature of enhance ASCII). Default behavior for an un-tagged file or stream is determined by the program, for example, `IBM Open Enterprise SDK for Python <https://www.ibm.com/products/open-enterprise-python-zos>`__ defaults to the UTF-8 encoding.

  The ansible core engine is unaware of tags in z/OS UNIX and thus processes all data as either binary or text encoded in UTF-8. This means that data sent to remote z/OS nodes is encoded in UTF-8 and un-tagged. The z/OS UNIX remote shell defaults to an EBCDIC encoding for un-tagged data streams. This mismatch in data encodings can be resolved with the ``PYTHONSTDINENCODING`` environment variable, which tags the pipe with the encoding specified. File and pipe tags are used for automatic conversion between ASCII and EBCDIC.

  When Ansible pipelining is enabled (`see the config option here <https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-pipelining>`_), Ansible passes any module code to the remote target node through python's stdin pipe and runs it in all in a single call. For more details on pipelining, see: :ref:`flow_pipelining`.

  Include the following in the environment for any tasks performed on z/OS target nodes. The value should be the local encoding used by the z/OS UNIX shell of the remote target.

  .. code-block:: yaml

    PYTHONSTDINENCODING: "cp1047"

  When Ansible pipelining is enabled but the ``PYTHONSTDINENCODING`` property is not correctly set, the following error may result. Note, the ``'\x81'`` below may vary based on the target user and host:

  .. error::
    SyntaxError: Non-UTF-8 code starting with '\\x81' in file <stdin> on line 1, but no encoding declared; see https://peps.python.org/pep-0263/ for details






How do I tell Ansible where IBM Open SDK for python is?
-------------------------------------------------------

* Ansible requires a python interpreter to execute modules on the remote host, and checks for it at the ‘default’ path ``/usr/bin/python``.

  | On z/OS, the Python 3 interpreter (from `IBM Open Enterprise SDK for Python <https://www.ibm.com/products/open-enterprise-python-zos>`_) is often installed to a different path, typically something like: 
  | ``<path-to-python>/usr/lpp/cyp/v3r12/pyz``.

  The path to the python interpreter can be configured with the Ansible inventory variable ``ansible_python_interpreter``.
  For example:

  .. code-block:: ini

    zos1 ansible_python_interpreter:/python/3.12/usr/lpp/cyp/v3r12/pyz

  When the path to the python interpreter is not found in the default location on the target host, the following error may result:

  .. error::
    /usr/bin/python: FSUM7351 not found

  For more details, see: :ref:`python_interpreters`.


Using SSH keys
--------------
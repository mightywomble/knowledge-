---
description: This is a set of challenges for an interview.
---

# Linux Test

### Instructions for interviewee



You will be dropped into a Virtual machine with a selection of issues wrong with it. Your task is to perform the instructions we give you then locate and rectify any issues you encounter. Do each step in order only. You are allowed to use the internet and or man pages to help you solve the issue. Please talk us through what you are thinking as you troubleshoot the issue. We are interested to see if you are able to diagnose and rectify these issues and we are not rating you based on 'best practice' or 'security'

* Change the password for user 'gilbert'.
* Switch user to 'gilbert'
* There is a file called '\~' in the home directory, delete it.
* Create an SSH key for the user 'gilbert', add the public key to authorized\_keys and test
* Install httpd
* start httpd and verify functionality
* there is a second disk attached to the system. format it with ext4 filesystem, create a persistant mount, mount it and create a file
* run the mystery application /usr/local/bin/pbcak
* identify why your mount no longer works and attempt to recover it.

Your role is for each of the steps above, to:

1. What commands did you use?
2. If there was a problem, what was wrong?
3. How did you solve the problem?

{% embed url="https://github.com/mightywomble/linuxtest" %}

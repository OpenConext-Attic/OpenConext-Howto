# OBSOLETE, archive only.

*For up to date installation documentation, see: https://github.com/OpenConext/OpenConext-deploy/wiki*

# OpenConext-Howto
OpenConext is developed on CentOS 6, soon 7 and the easy install requires a compatible setup on the target server. This manual however explores the journey of installing all the required parts on a fresh Ubuntu Xenial 16.04 LTS server. The purpose of the excercise being two-fold: help those unwilling or unable to fullfil the CentOS requirements and provide a thorough understanding of the actions "magically" done by the Ansible installation by executing them step-by-step, by hand.

Bear in mind that a normal OpenConext installation heavily relies on it's own technology for authentication. Which means most management interfaces are protected by SAML logins. For the sake of simplicity and speed I skipped this part of the configuration sometimes in order to have a working proof of concept without the need to create trivial authentication scenario's. Given the examples in this documentation it shouldn't be too difficult to switch to SAML login (or any other form of authentication) after initial setup.

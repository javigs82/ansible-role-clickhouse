# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.7.1] - 2021-01-13
### Add
 - fix molecule: add vagrant networking hostname option to avoid errors in naming resolution

## [1.7.0] - 2020-12-28
### Add
 - replace zookeeper rol in molecule

## [1.6.1] - 2020-12-11
### Fix
 - add listen default host as var instead of defaults

## [1.6.0] - 2020-12-11
### Add
 - replace custom config avoiding override default config

## [1.5.0] - 2020-12-11
### Add
 - allow delete users


## [1.4.0] - 2020-12-10
### Add
 - replace display_name by cluster_name var

## [1.3.1] - 2020-12-10
### Fix
 - fix zookeeper config file
 - replace systemctl by service

## [1.3.0] - 2020-12-05
### Add
- Improve docu and variable precedence. Complete changelog
- Move some defaults to vars for consistency

## [1.2.2] - 2020-12-05
### Fix
- How remote-server.xml.j2 is built has been fixed

## [1.2.1] - 2020-12-05
### Fix
- Set proper remote server configuration

## [1.2.0] - 2020-12-04
### Added
- Set cluster based on ansible inventory patterns and regex: All built on top of hostname

## [v1.1.1] - 2020-11-30
### Added
- Set cluster with N shards and M replicas per each one using vars with static DNS

# Hive 引擎介绍
## Hive On Tez
基本概述

Hive支持底层运算引擎更改为Tez，以支持更复杂的DAG运算，提供更优秀的性能。Tez可以将多个Job整合为一个Job进行运算，从而支持复杂DAG作业。
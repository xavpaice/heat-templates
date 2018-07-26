# Performance testing volumes inside instances

## Make a cluster of hosts to work with

Using Heat, we can deploy an environment with cbt (https://github.com/ceph/cbt)
installed and ready to go.

The environment has one jumpbox, named 'jumpbox' to use for access, plus a
number of hosts as fio runners, each with a volume attached - the number is
controlled by the num_instances parameter to the hot template.

Here's an example, in this case, I am using just the defaults which you'll find
in cbt_runner.yaml, and using one particular volume type.

```
$ openstack volume type list
+--------------------------------------+--------------+-----------+
| ID                                   | Name         | Is Public |
+--------------------------------------+--------------+-----------+
| 6a98c9d2-d1c4-4c4e-b565-acf2564d24cb | NHP 15K IOPS | True      |
| c5317077-8bf4-434c-9dfa-ad0df80dd5ca | NHP 10K IOPS | True      |
| abd1e359-e8c8-4b23-9be4-bbdac38d9e52 | NHP 5K IOPS  | True      |
| f01d9ae2-7132-4278-b330-151eebdaf487 | NBS          | True      |
+--------------------------------------+--------------+-----------+

openstack stack create --wait -t serverswithvolumes.yaml --parameter vol_type='NHP 15K IOPS' storpool-2

2018-07-06 00:42:01Z [storpool-2]: CREATE_IN_PROGRESS  Stack CREATE started
2018-07-06 00:42:01Z [storpool-2.router]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:02Z [storpool-2.cbt_keypair]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:03Z [storpool-2.cbt_keypair]: CREATE_COMPLETE  state changed
2018-07-06 00:42:03Z [storpool-2.securitygroup]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:03Z [storpool-2.router]: CREATE_COMPLETE  state changed
2018-07-06 00:42:04Z [storpool-2.securitygroup]: CREATE_COMPLETE  state changed
2018-07-06 00:42:04Z [storpool-2.private_net]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:05Z [storpool-2.private_net]: CREATE_COMPLETE  state changed
2018-07-06 00:42:05Z [storpool-2.subnet]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:05Z [storpool-2.server_group]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:06Z [storpool-2.subnet]: CREATE_COMPLETE  state changed
2018-07-06 00:42:06Z [storpool-2.router_interface]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:42:10Z [storpool-2.router_interface]: CREATE_COMPLETE  state changed
2018-07-06 00:42:54Z [storpool-2.server_group]: CREATE_COMPLETE  state changed
2018-07-06 00:42:54Z [storpool-2.jumpbox]: CREATE_IN_PROGRESS  state changed
2018-07-06 00:43:43Z [storpool-2.jumpbox]: CREATE_COMPLETE  state changed
2018-07-06 00:43:43Z [storpool-2]: CREATE_COMPLETE  Stack CREATE completed successfully
+---------------------+----------------------------------------------------------------------------------------------------------------------------+
| Field               | Value                                                                                                                      |
+---------------------+----------------------------------------------------------------------------------------------------------------------------+
| id                  | 7bb8a37d-7e64-4f8c-93bb-31864b6501de                                                                                       |
| stack_name          | storpool-2                                                                                                                 |
| description         | A HOT template that creates a group of VMs with Cinder volumes attached. Once created, it does nothing with the instances. |
|                     |                                                                                                                            |
| creation_time       | 2018-07-06T00:42:00Z                                                                                                       |
| updated_time        | None                                                                                                                       |
| stack_status        | CREATE_COMPLETE                                                                                                            |
| stack_status_reason | Stack CREATE completed successfully                                                                                        |
+---------------------+----------------------------------------------------------------------------------------------------------------------------+
```

Now we can go about giving the jumpbox a floating IP:

```
$ openstack server list
+--------------------------------------+-------------------------------------------------------+--------+---------------------------+------------------+--------+
| ID                                   | Name                                                  | Status | Networks                  | Image            | Flavor |
+--------------------------------------+-------------------------------------------------------+--------+---------------------------+------------------+--------+
| d99ed8af-7408-4414-b995-ab22424778ea | jumpbox                                               | ACTIVE | web_private_net=10.4.2.18 | Ubuntu 18.04 LTS | SM4.2  |
| 38d37dd9-16d6-4f21-b21c-00a6eeac859c | st-niqoo2y5yl-2-64m24oesmrqp-my_instance-eccl77ky2jzn | ACTIVE | web_private_net=10.4.2.23 | Ubuntu 18.04 LTS | SM4.2  |
| b6e433c7-2be4-4bfa-be63-5a49cd014cd1 | st-niqoo2y5yl-1-dxbss6j4l6gc-my_instance-se3hhwudeduk | ACTIVE | web_private_net=10.4.2.22 | Ubuntu 18.04 LTS | SM4.2  |
| f089654b-d713-46ee-a8d7-9f7d0ac5e7d2 | st-niqoo2y5yl-0-keg66pniylmm-my_instance-q5lpx7nkmi6v | ACTIVE | web_private_net=10.4.2.14 | Ubuntu 18.04 LTS | SM4.2  |

$ openstack floating ip list
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| ID                                   | Floating IP Address | Fixed IP Address | Port                                 | Floating Network                     | Project                          |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| 16d0fccf-7aaf-4c0f-8c99-7fc7d29b9f5b | 103.93.52.18        | None             | None                                 | fa2c3b23-5f25-4ab1-b06b-6edc405ec323 | 51b458e957be4e5b9212bb1e1f10c8cb |

jujumanage@jkt01z00infr001:~/hot$ openstack port list |grep '10.4.2.18'
| 598b1525-d904-45ec-8c56-8bc51b2f6918 | jumpbox-port-0                                    | fa:16:3e:6a:5e:b3 | ip_address='10.4.2.18', subnet_id='86faeeb1-711f-421d-a743-0d70d2dabbc9'      | ACTIVE |
jujumanage@jkt01z00infr001:~/hot$ openstack floating ip set --port 598b1525-d904-45ec-8c56-8bc51b2f6918 103.93.52.18


```

Next, check that the build has completed - it takes a few mins.

```

$ nova console-log jumpbox

<snip a ton>
[  213.469944] cloud-init[1005]: Provision done: 2018-07-06 00:47:27.511262
[  213.471046] cloud-init[1005]: Cloud-init v. 18.2 running 'modules:final' at Fri, 06 Jul 2018 00:44:46 +0000. Up 49.06 seconds.
[  213.471598] cloud-init[1005]: Cloud-init v. 18.2 finished at Fri, 06 Jul 2018 00:47:30 +0000. Datasource DataSourceOpenStack [net,ver=2].  Up 213.31 seconds
```
Get the IPs of the created hosts
```
$ openstack stack output show storpool-2 server_group_addresses
+--------------+--------------------------------------------+
| Field        | Value                                      |
+--------------+--------------------------------------------+
| description  | No description given                       |
| output_key   | server_group_addresses                     |
| output_value | [u'10.4.2.14', u'10.4.2.22', u'10.4.2.23'] |
+--------------+--------------------------------------------+
```


Log in and run cbt:

```
$ ssh 103.93.52.18

$ cd /home/ubuntu/cbt
$ sed -i "s/replaceme/['10.4.2.14', '10.4.2.22', '10.4.2.23']/" /home/ubuntu/cbt_raw.yaml
./cbt.py -a /home/ubuntu ../cbt_raw.yaml
```

The results of these tests should wind up living in /home/ubuntu/results

Copy those off to a host where you can analyse them, and make changes as
appropriate.

Lastly, clean up:
```
openstack stack delete storpool-2
```

If you wanted to make some graphs, here's some hints to get started:

for i in $(find . -type f -name '*10.4.2.*') ; do outfile=$(echo $i | awk -F'.' 'BEGIN { OFS = FS }; NF { NF -= 6 }; 1') ; cat $i >>${outfile}.log ; done
for i in $(find . -name '*.log') ; do sort -n -o $i $i ; done
for filename in $(find . -name '*_bw.log') ; do dir=$(dirname $filename) ; pushd $dir ; fio2gnuplot -g -t $dir -p '*_bw.*.log.*' -t ${dir}-bw -o ~/tmp/biznet/perf-coll; popd ; done


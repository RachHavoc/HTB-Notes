
2. Start VM Group 2. A server is running on HRSHARES port 4242. Download the **ssh_local_client** binary from the _Resources_ section. If you're using the _aarch64_build of Kali, download the **ssh_local_client_aarch64** binary. Create an SSH local port forward on CONFLUENCE01, which will let you run the binary from your Kali machine against the server on HRSHARES and retrieve the flag.

Note: the source files used to build the **ssh_local_client** and **ssh_local_client_aarch64**binaries can be downloaded from **/exercises/client_source.zip** on CONFLUENCE01.

CONFLUENCE01 -**192.168.194.63**
HRSHARES - **172.16.194.217**
PGDATABASE01 - **10.4.194.215**

`ssh -N -L 0.0.0.0:4455:172.16.194.217:4242 database_admin@10.4.194.215`
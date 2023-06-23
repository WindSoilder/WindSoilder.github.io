# tcpdump
Currently, I use `tcpdump -i ens0 -w output.cap port 18089` mostly.

Here are some simple examples:
1. normally usage example: `tcpdump -i <network_interface>`, example: `tcpdump -i ens0`
2. output to file: `tcpdump -i <network_interface> -w <output_file>`, example: `tcpdump -i ens0 -w output.cap`
3. capture with specific port: `tcpdump -i <network_interface> -w <output_file> port <port_number>`, example: `tcpdump -i ens0 -w output.cap port 18089`.

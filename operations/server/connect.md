#Connecting to a remote server

Connect to the server using `ssh`:

```
ssh [user]@[ip-address] -p [port]
```

Copy file to server using `scp`:

```
scp -P [port] path/to/file [user]@[ip-address]:path/to/file 
```

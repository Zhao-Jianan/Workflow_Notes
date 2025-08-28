## Download
```
scp -r ajhz839@hpc5.bioeng.auckland.ac.nz:/path/to/hpc_folder/ "C:\local\destination\"
```


## Upload
```
scp -r "C:\local\myfolder"  ajhz839@hpc5.bioeng.auckland.ac.nz  :/path/to/hpc_folder/
```


## Copy to network drive

### Windows
```
Copy-Item -Path "C:\local\myfolder\*" -Destination "Z:\target_folder\" -Recurse
```
### Linux
```
cp -r ~/local/myfolder/* /mnt/zdrive/target_folder/
```


## Copy from network drive

### Windows
```
Copy-Item -Path "Z:\source_folder\*" -Destination "C:\local\target_folder\" -Recurse
```
### Linux
```
cp -r /mnt/zdrive/source_folder/* ~/local/target_folder/
```
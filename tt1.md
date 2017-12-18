
To build image with UNCLOT:
```
osv @ jc-ipbypass-1-20171129-1056---opendata == jc-ipbypass-1-20171129-1056 == be6ba0b22591ded3fb3cff793ab36dede4fe1ce2
 (some code cleanup is expected)

cat config.json 
{
   "modules": {
        "osvinit": {
            "type": "direct-dir",
            "path": "${OSV_BASE}/modules/cloud-init/"
        },
    "repositories": [
        "${OSV_BASE}/apps",
        "${OSV_BASE}/modules"
        , "${OSV_BASE}/../mike-apps"
        , "${OSV_BASE}/../openfoam-cases"
    ]
   },
   "default": [ "cloud-init", "httpserver" ]
}
```

Build image with nginx and wrk:
```
./scripts/build mode=release image=cli,nginx,wrk nfs=true -j8
```

Build image with OpenFOAM and test data:
```
./scripts/build mode=release image=cli,cloud-init,udpping,open-mpi,OpenFOAM,mik3d_15min-2cpu,mik3d_15min-4cpu,mik3d_15min-8cpu,osu-micro-benchmarks nfs=true -j8
```

To start 2 VMs on smae host:
```
IMG=unclot-with-nginx-wrk---usr.img
# or
# IMG=unclot-with-openfoam---usr.img

/bin/rm /dev/shm/ivshmem
/bin/cp $IMG work/vm0.img && ./scripts/run.py --novnc --gdb 1234 -nv -c4 --pass-args '--device ivshmem,shm=ivshmem,size=1024M' -e "/cli/cli.so" --mac 52:54:00:12:34:56 -V -i work/vm0.img | tee aa0
/bin/cp $IMG work/vm1.img && ./scripts/run.py --novnc --gdb 1235 -nv -c4 --pass-args '--device ivshmem,shm=ivshmem,size=1024M' -e "/cli/cli.so" --mac 52:54:00:12:34:57 -V -i work/vm1.img | tee aa1
```

To run nginx and wrk case:
```
osv_run 192.168.122.90 "/nginx.so -c /nginx/conf/nginx.conf"; sleep 8; curl -X PUT http://192.168.122.91:8000/app/?command="$(rawurlencode "/wrk -c2 -d10 -t2 http://192.168.122.90:80/basic_status")"\&new_program=1
```

osv_run bash function is defined as:
```
# http://stackoverflow.com/questions/296536/how-to-urlencode-data-for-curl-command
rawurlencode() {
  local string="${1}"
  local strlen=${#string}
  local encoded=""
  local pos c o

  for (( pos=0 ; pos<strlen ; pos++ )); do
     c=${string:$pos:1}
     case "$c" in
        [-_.~a-zA-Z0-9] ) o="${c}" ;;
        * )               printf -v o '%%%02x' "'$c"
     esac
     encoded+="${o}"
  done
  echo "${encoded}"    # You can either set a return variable (FASTER) 
  REPLY="${encoded}"   #+or echo the result (EASIER)... or both... :p
}

osv_run() {
IP=$1
CMD="$2"
curl -X PUT http://$IP:8000/app/?command="$(rawurlencode "$CMD")"
}
```

To run OpenFOAM case:
```
curl -X PUT http://192.168.122.90:8000/app/?command="$(rawurlencode "/usr/bin/mpirun --allow-run-as-root -H 192.168.122.90,192.168.122.91 -np 4 -x TERM=xterm -x MPI_BUFFER_SIZE=20100100 -x WM_PROJECT_DIR=/openfoam/ /usr/bin/simpleFoam.so -case /openfoam/mik3d_15min-4cpu/ -parallel")"\&new_program=1
```

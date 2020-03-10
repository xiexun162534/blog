---
title: Generate Linux Kernel LLVM Bitcode with Custom Optimization
date: 2020-03-08 22:42:22
tags:
- LLVM
- Linux
---

In [Compile Linux Kernel to LLVM Bitcode](/compile-linux-llvm.html) we use wllvm to generate the LLVM bitcode of Linux kernel. when analyzing the kernel, we sometimes need to disable some optimization passes. However, it's hard to do with clang. First compiling it to `-O0` IR and then applying some optimization passes on it seems easier.

The optimization level of the LLVM IR is `-O2` by default. If we need an `-O0` LLVM IR, we need to modify the kernel compilation flag.
``` shell
make KCFLAG='-O0' CC=wllvm
```
However, it doesn't work because some kernel code cannot be compiled with `-O0`.

Fortunately, we have another way to get our `-O0` LLVM IR.

When wllvm generates LLVM bitcode (`.filename.o.bc`) for a C file, it also generates the command to compile the file in `.filename.o.cmd`. We can get the command by:
``` bash
path='/path/to/our/output'
awk_cmd='{ 
s = ""; 
ofile = 0;
for (i = 3; i <= NF; i++)
{
   if (ofile == 1)
   {
      s = s "'"$path"'" " ";
      ofile = 0;
   }
   else
   {
      if ($i == "-o")
         ofile = 1;
      s = s $i " ";
   }
}
print s }'
cmd_line=`head -n 1 $1 | awk "$awk_cmd"`
```
The output file path will be replaced by the path we provide.

This command generates native code. To get LLVM bitcode, we need to add `-emit-llvm`. To disable optimization, we need to add `-mllvm -disable-llvm-optzns`.

After generating `-O0` bitcode, we can optimize it with any passes we want. Finally link all bitcodes we generate and we get a kernel bitcode with custom optimization.

Below is an example script:

``` bash
#!/bin/bash


llvm_linker=llvm-link

function get_cmd
{
    path="$2"
    
    # change output file name
    awk_cmd='{ 
    s = ""; 
    ofile = 0;
    for (i = 3; i <= NF; i++)
    {
      if (ofile == 1)
      {
        s = s "'"$path"'" " ";
        ofile = 0;
      }
      else
      {
        if ($i == "-o")
           ofile = 1;
        s = s $i " ";
      }
    }
    print s }'
    cmd_line=`head -n 1 $1 | awk "$awk_cmd"`
    
    # output LLVM IR
    cmd_line="${cmd_line} -emit-llvm"
    # disable opt
    cmd_line="${cmd_line} -mllvm -disable-llvm-optzns"
    echo $cmd_line
    return
}

function compile_opt
{
    cmd_path="$1"
    basepath="$2"
    tmp_path="${basepath}.tmp.bc"
    output_path="${basepath}.opt.bc"
    cmd=`get_cmd "$cmd_path" "$tmp_path"`
    eval "$cmd" 2> /dev/null
    cmd_result=$?
    if [ ! -e "$tmp_path" ] || [ $cmd_result -ne 0 ]
    then
        exit 1
    fi
    opt -always-inline -inline -mem2reg -simplifycfg -instcombine "$tmp_path" -o "$output_path"
    cmd_result=$?
    if [ ! -e "$output_path" ] || [ $cmd_result -ne 0 ]
    then
        exit 1
    fi
}

while getopts ":l:" opt; do
    case $opt in
        l) llvm_linker=$OPTARG
           ;;
        \?) echo "Usage: -l <llvm-link>" 1>&2
            exit 1
            ;;
    esac
done
shift $((OPTIND-1))
from_file=$1
if [ -z "$from_file" ]
then
    echo "no input file" 1>&2
    exit 1
fi

extract-bc -l false -m $from_file
manifest="${from_file}.llvm.manifest"
if [ ! -r "$manifest" ]
then
    echo "no manifest file" 1>&2
    exit 1
fi

tmpmanifest="${from_file}.tmp.manifest"
echo -n "" > "$tmpmanifest"

cat "$manifest" |
    while read o_bc_name;
    do
        basename=`basename "${o_bc_name%.o.bc}"`
        # basename="${basename#.}"
        dir="${o_bc_name%/*}"
        basepath="${dir}/${basename}"
        if [ -z "$dir" ] && [ -z "$basename" ]
        then
            continue
        fi
        
        cmd_path="${basepath}.o.cmd"
        if [ ! -r "${cmd_path}" ]
        then
            echo "${cmd_path} not found"
            continue
        fi

        while [ `jobs | wc -l` -ge 8 ]
        do
            sleep 0.1
        done
        output_path="${basepath}.opt.bc"
        compile_opt "$cmd_path" "$basepath" &
        echo "$output_path"
        echo "$output_path" >> "${tmpmanifest}"
    done

while [ `jobs | wc -l` -ne 0 ]
do
    sleep 1
done

outputfile="${from_file}.opt.bc"
$llvm_linker -o "$outputfile" `cat "${tmpmanifest}"`

```

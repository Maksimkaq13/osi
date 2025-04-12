#!/bin/bash
snapshot_initial="io_snapshot_initial.txt"
snapshot_final="io_snapshot_final.txt"
delta_file="delta.txt"
output_file="max_io_processes.txt"
rm -f "$snapshot_initial" "$snapshot_final" "$delta_file" "$output_file"

for pid_dir in /proc/[0-9]*; do
    pid=$(basename "$pid_dir")
    io_file="$pid_dir/io"
    if [ -r "$io_file" ]; then
        read_bytes=$(grep '^read_bytes:' "$io_file" | awk '{print $2}')
        if [ -r "$pid_dir/cmdline" ]; then
            cmd=$(tr '\0' ' ' < "$pid_dir/cmdline")
            [ -z "$cmd" ] && cmd=$(cat "$pid_dir/comm")
        else
            cmd=""
        fi
        echo "$pid $read_bytes $cmd" >> "$snapshot_initial"
    fi
done

sleep 60

for pid_dir in /proc/[0-9]*; do
    pid=$(basename "$pid_dir")
    io_file="$pid_dir/io"
    if [ -r "$io_file" ]; then
        read_bytes=$(grep '^read_bytes:' "$io_file" | awk '{print $2}')
        if [ -r "$pid_dir/cmdline" ]; then
            cmd=$(tr '\0' ' ' < "$pid_dir/cmdline")
            [ -z "$cmd" ] && cmd=$(cat "$pid_dir/comm")
        else
            cmd=""
        fi
        echo "$pid $read_bytes $cmd" >> "$snapshot_final"
    fi
done

while read -r pid init_read rest; do
    final_line=$(grep "^$pid " "$snapshot_final")
    if [ -n "$final_line" ]; then
        final_read=$(echo "$final_line" | awk '{print $2}')
        diff=$((final_read - init_read))
        echo "$pid $diff $rest" >> "$delta_file"
    fi
done < "$snapshot_initial"

sort -k2 -n -r "$delta_file" | head -n 3 > "$output_file"
while read -r pid diff cmd; do
    echo "$pid : $cmd : $diff"
done < "$output_file"

# Scripts
# To extract file names
#!/bin/bash 

find /project/4180000.24/Previn/PND30_cut/ -type f -name "*.mp4" -exec basename {} \; | sed 's/\.mp4$//' > /project/4180000.24/Previn/PND30files.csv 
find /project/4180000.24/Previn/PND70_cut/ -type f -name "*.mp4" -exec basename {} \; | sed 's/\.mp4$//' > /project/4180000.24/Previn/PN7D0files.csv 


# For automatic merging
#!/bin/bash
cd <set working directory>
`# Iterate over each .mp4 file in the folder`
for file in *.mp4; do
    # Check if the file name ends with P1 or P2
    if [[ "$file" =~ ^(.*)(P[12])\.mp4$ ]]; then
        # Extract the basename without the part number
        basename="${BASH_REMATCH[1]}"
        # Extract the part number
        part_number="${BASH_REMATCH[2]}"
        # Determine the counterpart part number
        counterpart_part_number="P$(((${part_number//[^0-9]/} % 2) + 1))"

        # Check if both parts exist
        part1_file="${basename}P1.mp4"
        part2_file="${basename}P2.mp4"
        if [ -f "$part1_file" ] && [ -f "$part2_file" ]; then
            # Check if both parts contain audio streams
            audio1=$(ffprobe -v error -select_streams a:0 -show_entries stream=codec_type -of csv=p=0 "$part1_file")
            audio2=$(ffprobe -v error -select_streams a:0 -show_entries stream=codec_type -of csv=p=0 "$part2_file")

            if [ -n "$audio1" ] && [ -n "$audio2" ]; then
                # Merge both parts with video and audio streams
                ffmpeg -i "$part1_file" -i "$part2_file" -filter_complex "[0:v][0:a][1:v][1:a]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" "${basename}_merged.mp4" -y
            else
                # Merge both parts with video streams only
                ffmpeg -i "$part1_file" -i "$part2_file" -filter_complex "[0:v][1:v]concat=n=2:v=1:a=0[v]" -map "[v]" "${basename}_merged.mp4" -y
            fi

            # Optionally, delete the original parts
            # rm "$part1_file" "$part2_file"
            echo "Merged $part1_file and $part2_file into ${basename}_merged.mp4"
        elif [ -f "$counterpart_part_number" ]; then
            # Merge the two parts using ffmpeg (order reversed)
            ffmpeg -i "$counterpart_part_number" -i "$file" -filter_complex "[0:v][1:v]concat=n=2:v=1:a=0[v]" -map "[v]" "${basename}_merged.mp4" -y
            # Optionally, delete the original parts
            # rm "$counterpart_part_number" "$file"
            echo "Merged $file and $counterpart_part_number into ${basename}_merged.mp4"
        else
            echo "Error: Corresponding part not found for $file"
        fi
    fi
done


# For automatic trimming
#!/bin/bash

`# Path to the CSV file`
csv_file="/project/4180000.24/Previn/trimsheet_PND30.csv"

`# Path to the folder containing the .mp4 files`
video_folder="/project/4180000.24/Previn/PND30_cut/"
out

`# Read the CSV file and process each row`
awk -F ',' 'NR > 1 { print $1,$2,$3 }' "$csv_file" | while read -r start end video_name; do
    echo "Processing video: "$video_name
    echo "Start time: "$start
    echo "End time: "$end
    
    # Construct the full file paths
    input=${video_folder}${video_name}
    output=${video_folder}${video_name}_cut
    

    # Check if input file exists
    if [ ! -f $input ]; then
        echo "Input file $input not found."
        continue
    fi

    # Run ffmpeg to cut the video
    ffmpeg -i ${input}.mp4 -ss ${start} -to ${end} -c:v copy -c:a copy ${output}.mp4

done


# For cropping
#!/bin/bash

ffmpeg -i in.mp4 -vf "crop=out_w:out_h:x:y" out.mp4

large videos: x=393 y=180, w=508 h=328
small videos: x=230 y=127, w=557 h=300

cd /project/4180000.24/Previn/wessel/

in_1="/project/4180000.24/Previn/wessel/large_videos/"
in_2="/project/4180000.24/Previn/wessel/small_videos/"

out_1="/project/4180000.24/Previn/wessel/large_cropped/"
out_2="/project/4180000.24/Previn/wessel/small_cropped/"

for file in "$in_1"/*; do
	basename=$(basename "$file")
	filename="${basename%.*}"
	echo "processing video ${filename}"

	ffmpeg -i ${file} -vf "crop=508:328:393:180" ${out_1}/${filename}_cropped.mp4 
done

for file in "$in_2"/*; do
	basename=$(basename "$file")
	filename="${basename%.*}"
	echo "processing video ${filename}"

	ffmpeg -i ${file} -vf "crop=557:300:230:127" ${out_2}/${filename}_cropped.mp4 
done

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

# For quantifying social interaction
#!/usr/bin/env python3

#`Only focus on the x and y coordinates of the nose and tailabse
#`Need to know the number of frames where they meet the condition that they are not further apart than 5 pixels

#`Import libraries
import pandas as pd
import numpy as np

#`Load data
df = pd.read_csv("/Users/macair/Desktop/finalising internship/Coordinates.csv", header=None)

#`To easily reach to the data, create a hierarchical name by combining the first few rows
headers= df.iloc[:4].apply(lambda x: '_'.join(x), axis=0)
data = df.iloc[4:] # to extract the numeric data
data.columns = headers
data.reset_index(drop=True, inplace=True)

#`Filter the original DF (Coordinates.csv) to filter out all the columns you don't need
#`Only take the nose and tailbase columns
selected_columns = [
    "DLC_resnet50_test123May13shuffle5_100000_individual1_nose_x", 
    "DLC_resnet50_test123May13shuffle5_100000_individual1_nose_y",
    "DLC_resnet50_test123May13shuffle5_100000_individual1_tailbase_x",
    "DLC_resnet50_test123May13shuffle5_100000_individual1_tailbase_y",
    "DLC_resnet50_test123May13shuffle5_100000_individual2_nose_x", 
    "DLC_resnet50_test123May13shuffle5_100000_individual2_nose_y",
    "DLC_resnet50_test123May13shuffle5_100000_individual2_tailbase_x",
    "DLC_resnet50_test123May13shuffle5_100000_individual2_tailbase_y"]
filtered_df= data[selected_columns]

for col in selected_columns:
    filtered_df[col] = pd.to_numeric(filtered_df[col], errors='coerce') 
#`pd.to.numeric is to ensure the values are numeric
#`if they are not converted to numeric values, error='coerce' will return NaNI

#`To calculate the distance, use the following formula (Pythagorean theorem):
#`d=√((x_2-x_1)²+(y_2-y_1)²)

#`Add 2 columns where for every frame (for every row containing data) you calculate the difference in x and y
#`Do this for individual 1 sniffing individual 2's butt
filtered_df['Δx1'] = filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual1_nose_x"] - filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual2_tailbase_x"]
filtered_df['Δy1'] = filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual1_nose_y"] - filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual2_tailbase_y"]
#`Also for invidual 2 sniffing individual 1's butt
filtered_df['Δx2'] = filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual2_nose_x"] - filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual1_tailbase_x"]
filtered_df['Δy2'] = filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual2_nose_y"] - filtered_df["DLC_resnet50_test123May13shuffle5_100000_individual1_tailbase_y"]

#`Using these columns calculate the absolute distance in pixels (square root)
#`Do this for ind1 sniffing indv2
filtered_df['absolute_distance1'] = np.sqrt((filtered_df['Δx1']**2) + (filtered_df['Δy1']**2))
#`Also for ind2 sniffinf indv1
filtered_df['absolute_distance2'] = np.sqrt((filtered_df['Δx2']**2) + (filtered_df['Δy2']**2))

#`In a fourth column, apply a boolean func that says whether the absolute distance is < threshold via returning True or False
#`This gives the exact number of frames where the distance between nose and tailbase was < threshold, for both cases: ind1 sniffing ind2, ind2 sniffing indv1
#`Interaction threshold in pixels, conversion: 1 pixel = 0.1 cm & to be interacting < 2cm
interaction_threshold = 20  
filtered_df['interaction1'] = filtered_df['absolute_distance1'] <= interaction_threshold
filtered_df['interaction2'] = filtered_df['absolute_distance2'] <= interaction_threshold

#`Find the number of frames where interaction occurs
interaction_frames1 = filtered_df['interaction1'].sum()
interaction_frames2 = filtered_df['interaction2'].sum()

#`Divide by 25 to get the interaction time in seconds (25 frames per second)
interaction_time1 = interaction_frames1/25
interaction_time2 = interaction_frames2/25

#`Add the two interaction times to find the total interaction time
total_interaction_time = interaction_time1 + interaction_time2

print("Interaction Time (ind1 sniffing ind2:", interaction_time1, "seconds")
print("Interaction Time (ind2 sniffing ind1:", interaction_time2, "seconds")
print("Total Interaction Time:", total_interaction_time, "seconds")
#`Getting the individual interaction times also help differentiate between the genotypes of the mice

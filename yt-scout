#!/bin/bash

# depends ggrep, pcregrep, ffmpeg, youtube-dl
#                ^ will hopefully remove this soon :)

while getopts 'l:f:s:k' flag; do
    case "${flag}" in
    l) link=${OPTARG} ;;
    f) format=${OPTARG} ;;
    s) userSearchPattern=${OPTARG} ;;
    k) keepAllFiles=${OPTARG} ;;
    esac
done

searchPattern=$(echo "$userSearchPattern" | tr " " "\n")

searchPatternArray=($searchPattern)
# ^ referenced like ${searchPatternArray[n]}

dir="videoSearchTemp-$(date +%Y%m%d%H%M%S)"

echo && echo "Beginning subtitle download step" && echo

(exec youtube-dl --write-auto-sub --convert-subs=srt --skip-download -ciw -v ${link} -o "$TMPDIR//%(id)s")

echo && echo "Beginning video download and trim step" && echo
for subs in $(ls $TMPDIR/); do

    # https://stackoverflow.com/a/8260383
    id=$(echo "$link" | pcregrep -o1 '.*(?:youtu.be\/|v\/|u\/\w\/|embed\/|watch\?v=)([^#\&\?]*).*')


    # this one would work great. and it works fine on regex101. but for some reason pcregrep doesn't include all the matches
    # (exec cat $dir/$subs | pcregrep -M -o '(?<=Language: en\n\n).*(?= -->)|(?<=<c> )[^<]*(?=<\/c>)|(?<=%\n)^[ ]{0,1}\w++$(?=\n)|^[^\n<]++(?=<)|(?<=<)[^<>]*(?=><c>)|(?<= \n\n)[^ ]*(?= --> )' >"$TMPDIR//$id-combined.txt")

    # this one is what i have to use instead. It's a bit janky because it returns empty lines, and then sed removes the empty lines. but it's the only way I could get it to work so yeah
    (exec cat $dir/$subs | pcregrep -Mo '(?<=Language: en)[\s\S]*?(?= -->)|(?<=<c> )[^<]*(?=<\/c>)|(?<=%\s)^[ ]{0,1}\w++$(?=\s[^a-zA-Z0-9])|^[^\s<]++(?=<)|(?<=<)[^<>]*(?=><c>)|(?<= )[^a-z][^ ]*(?= --> )' | sed '/^$/d' >"$TMPDIR//$id-combined.txt")

    # this gets a file you can actually search through
    (exec cat $dir/$subs | pcregrep -Mo '(?<=<c> )[^<]*(?=<\/c>)|(?<=%\n)^[ ]{0,1}\w++$(?=\n)|^[^\n<]++(?=<)' >"$TMPDIR//$id-words.txt")

    # download videos if they have matches
    countInVideo=$(grep -rohc "$searchPattern" "$TMPDIR//$id-words.txt")
    Echo "Found $countInVideo match(es) of $searchPattern in $id"

    if [ $(exec grep -rohc "$searchPattern" "$TMPDIR//$id-words.txt") -gt 0 ]; then
        echo "[youtube-dl] downloading $id..."

        # download the best seperate audio+video files, then combine. Less reliable
        # (exec youtube-dl -f 'bestvideo[ext=mp4]+bestaudio[ext=m4a]/bestvideo+bestaudio' --merge-output-format mp4 $id -o "$TMPDIR//%(id)s")
        # download the best single file. Pretty much guaranteed to work.
        (exec youtube-dl -f best[ext=mp4] $id -o "$TMPDIR//%(id)s.%(ext)s")

        splitSearchTerm=(${searchPattern// / })
        searchterm1=${splitSearchTerm[0]}

        # (exec grep -n ${splitSearchTerm[1]} $TMPDIR//$id-timecodes.txt | cut -f1 -d:)

        # for match in $(grep -n $searchPattern $TMPDIR//$id-words.txt | cut -f1 -d:); do
        #     echo $match
        # done

        numMatches=($(exec pcregrep -n $searchPattern $dir/$id-words.txt | pcregrep -o "^[0-9]{1,}"))

        for lineNumber in "${numMatches[@]}"; do

            echo "$lineNumber"

            ((startIndex = $lineNumber * 2 - 1))
            start=$(sed -n ${startIndex}p $TMPDIR//$id-combined.txt)

            ((endIndex = $lineNumber * 2 + 1))
            end=$(sed -n ${endIndex}p $TMPDIR//$id-combined.txt)

            # initialLineNumber=${timecodesArray[lineNumber-1]}
            # finalLineNumber=${timecodesArray[lineNumber]}

            echo "[ffmpeg] ($(echo ${numMatches[@]/$lineNumber//} | cut -d/ -f1 | wc -w | tr -d ' ')/${#numMatches[@]}) trimming $id.mp4 from $start to $end into $dir/$id-$lineNumber.mp4"

            # This one takes a long ass time. but it gives more reliable results
            (exec ffmpeg -hide_banner -loglevel error -i "$dir/$id.mp4" -ss $start -to $end -c:v libx264 -c:a aac "$dir/$id-$lineNumber.mp4")
            # This one is MUCH faster. But the file you get isnt always playable (?)
            # (exec ffmpeg -hide_banner -loglevel error -i "$dir/$id.mp4" -ss ${timecodesArray[lineNumber]} -to ${timecodesArray[lineNumber + 1]} -c:v copy -c:a copy "$dir/$id-$lineNumber.mp4")

        done

    fi

    echo "removing some temp files that are no longer needed..."
    # (exec rm "$TMPDIR//$id-words.txt" "$TMPDIR//$id-timecodes.txt" "$TMPDIR//$id.mp4" "$TMPDIR//$subs")

done

echo ""
echo "Beginning compilation step"
echo ""

for file in $(basename $dir/*.mp4); do
    echo file \'$file\' >>$dir/list.txt
done
(exec ffmpeg -hide_banner -loglevel error -f concat -safe 0 -i $dir/list.txt -c copy "\"$userSearchPattern\"\ Matches.mp4")

echo ""
echo "Removing the rest of the temp files..."
echo ""

echo "Done. Saved to $(readlink -f ./\"$userSearchPattern\"\ Matches.mp4)"
echo ""
# rm -r $dir

# videoName=`ls -rt $dir/$subs*`
# echo "$videoName"

# id=``

# (exec youtube-dl --write-auto-sub --convert-subs=srt --skip-download -ciw -v ${link} -o "$TMPDIR//%(id)s.%(ext)s")

# finds all timecode in .vtt:
#   (Has to have all the or statements (|) to get the timecodes at the very top and bottom too)
# .{1,}(?= -->.*\n )|.{1,}(?= -->.*\n.*\n )|(?<=<)[^>c]*(?=>)

# finds all subs in .vtt:
# ^[^<\n]*(?=<[0-9])|(?<=<c> )[^<]*

# Finds all timecodes and text in .vtt:
# ([^<>\n]*)(<\/c><|<\/c>|><c>|<)
# no spaces:
# ([^<>\n ]*)(<\/c><|<\/c>|><c>|<)

# trim the matches
# for videoDir in $(find "$dir" -type f | perl -lne 'print if -B'); do
#     echo $videoDir
#     # trim video
#     # (exec ffmpeg -hide_banner -loglevel error -i "$videoDir" -ss 00:00:03 -t 00:00:08 -async 1 "$dir/ffmpeg.mp4")
# done

# output
# (exec ffmpeg -i opening.mkv -i episode.mkv -i ending.mkv \
# -filter_complex "[0:v] [0:a] [1:v] [1:a] [2:v] [2:a] \
# concat=n=3:v=1:a=1 [v] [a]" \
# -map "[v]" -map "[a]" output.mkv)

# how this may work:
# 1. download all subtitles from a channel
#     a. output would ideally be just the video id
# 2. for each subtitle, download the video (by the video id provided by 1a)
#   a. maybe ask user for what format they want (1080p, 720, etc)? https://askubuntu.com/questions/486297/how-to-select-video-quality-from-youtube-dl
#   b. try to use the same format for each video, unless a format is unavalable. Then ask them which one to use instead. Press 'enter' to just use the next best one.
#   c. if a better format is avalable, ask if they want to use that instead ('enter' = yes)
# 3. then match for the specified text in the subtitles
# 4. for each match, run ffmpeg, trimming the video to each match
#    a. this will need to be down to the frame, which means frame rate must be determined. See https://superuser.com/questions/459313/how-to-cut-at-exact-frames-using-ffmpeg
# 5. combine all the ffmpeg clips into one somehow

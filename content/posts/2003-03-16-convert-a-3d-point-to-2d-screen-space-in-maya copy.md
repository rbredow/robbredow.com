---
author: Rob
categories:
- tech
date: "2003-03-16T23:14:52Z"
guid: /2006/02/convert-a-3d-point-to-2d-screen-space-in-maya/
id: 13
image: /images/2003/03/mel-screen-space.avif
tags:
- tech
- VFX
- Open Source
title: Convert a 3d point to 2d Screen Space in Maya
url: /2003/03/convert-a-3d-point-to-2d-screen-space-in-maya/
---

Very often you want to find the 2d coordinates on the screen for a given 3d point. Here’s a [little mel script](/resources/screenSpace.mel) that knows the projection math and takes care of generating the data in a “raw channel text file” format. From here, it’s easy to generate just about any ascii file format that a compositing program might be expecting.

I’ve seen people rendering out small green tracking markers on their 3d objects to later track them in 2d to get approximately the same results. The computer can save you these steps and get you the exact result much quicker. To quote Homer Simpson, “Computer’s can do that!"

<audio controls>
  <source src="/resources/computerscandothat.wav" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>
<br/>
<br/>  

```

// Rob Bredow
// rob (at) 185vfx.com
// http://www.185vfx.com/
// Copyright 3/2002 Rob Bredow, All Rights Reserved
// 
// Converts currently selected point or object from 3d space to 
// screen space for use in a compositing package.
//
// Writes out "normalized" 2d coordinates so multiply the x by the
// width of your image and y by the height to get pixels
//
// Assumes your camera is called "cam_main"


// Get a matrix
proc matrix screenSpaceGetMatrix(string $attr){
  float $v[]=`getAttr $attr`;
  matrix $mat[4][4]=>;
 return $mat;
}

// Multiply the vector v by the 4x4 matrix m, this is probably
// already in mel but I cant find it.
proc vector screenSpaceVecMult(vector $v, matrix $m){
  matrix $v1[1][4]=>;
  matrix $v2[1][4]=$v1*$m;
  return >;
}

global proc int screenSpace()
{
  // get the currently selected point
  string $dumpList[] = `ls -sl`;
  int $argc = size($dumpList);

  if ($argc != 1)
  {
    confirmDialog -t "screenSpace Usage" -m "To use, select a point on a geometry or a single object to write";
    return -1;
  }
  
  string $dumpPt = $dumpList[0];

  // Make $dumpPt a unix friendly name
  string $name = $dumpPt;
  while ( match("\\[",$name) != "" ) {
    $name = `substitute "\\[" $name "_"`;
  }
  while ( match("\\]",$name) != "" ) {
    $name = `substitute "\\]" $name ""`;
  }
  while ( match("\\.",$name) != "" ) {
    $name = `substitute "\\." $name "_"`;
  }

  // Find the frame range
  float $fs = `playbackOptions -q -min`;
  float $fe = `playbackOptions -q -max`;
  string $verify = "Will create " + $name + "_screenSpace.cmp\n" +
                   "For the frame range of " + $fs + "-" + $fe + "\n" +
                   "The camera used will be cam_main\n";
 
  if (`confirmDialog -t "screenSpace Verify" -m $verify -b "Dump" -b "Cancel"` == "Cancel")
    return -2;

  print ("Dumping selection...\n");

  string $pointWsFile = $name+"_screenSpace.cmp";
  int $outFileId = fopen($pointWsFile,"w");

  if ($outFileId == 0) {
    print ("Could not open output file " + $pointWsFile);
    return -1;
  }

  string $line;

  int $f;
  float $tx[],$ty[],$tz[];
  for ($f=$fs;$f>;

    // Grab the worldInverseMatrix from cam_main
    matrix $cam_mat[4][4] = screenSpaceGetMatrix("cam_main.worldInverseMatrix");

    // Multiply the point by that matrix
    vector $ptVecCs = screenSpaceVecMult($ptVecWs,$cam_mat);

    // Adjust the point's position for the camera perspective
    float $hfv = `camera -q -hfv cam_main`;
    float $ptx = (($ptVecCs.x/(-$ptVecCs.z))/tand($hfv/2))/2.0+.5;
    float $vfv = `camera -q -vfv cam_main`;
    float $pty = (($ptVecCs.y/(-$ptVecCs.z))/tand($vfv/2))/2.0+.5;

    float $ptz = $ptVecCs.z;

    $line = $ptx + " " + $pty + " " + "\n";
    fprint $outFileId $line; 
  }
 
  fclose $outFileId;

  return 1;
}
```

Here’s an [alternative version](http://185vfx.com/dl/scrSpace02.mel) submitted by Andrew Roberts that looks to be more complete in every way. It builds a simple interface for the user and handles non-square pixels (NTSC) accurately.
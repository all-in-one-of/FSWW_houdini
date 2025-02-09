Fundemental Solutions For Water Wave Animation
------------------------------------------------------------------------------

This code implements a simplified version the method described in

    "Fundemental Solutions For Water Wave Animation"
    by Camille Schreck, Christian Hafner and Chris Wojtan
    published in ACM Transactions on Graphics (SIGGRAPH 2019)

This code is not particularly well-tested or optimized, but it can be used to
reproduce (some) of the examples in the paper. This is not a GPU or multithreaded version.
The code depends on Houdini. We have only tested the code using Houdini FX 17.0.459 under
Ubuntu 18.04.

To use the code, you first have to compile the plug-in. A Makefile is provided
in the root directory. Before you can compile the code, you need to
load in the HDK environment variables.

    $ pushd /opt/hfs17.0.459
    $ source houdini_setup
    $ popd

Then you can compile the code.

    $ make

Once the plug-in is installed in your home directory (there should be files like "SOP_Create_Source.so" in
~/houdini17.0/dso/) you can load up the included Houdini scene
file.

    $ houdini fsww_periodic.hipnc 
or
    $ houdini fsww_periodic.hipnc
(Note that the aperiodic method is still in a very early version and not tested)
------------------------------------------------------------------------------
Camille Schreck <camille.schreck@ist.ac.at
Last updated 2019-08-07

--------------------------------------------------------------------------------
Implementation
--------------------------------------------------------------------------------
If you want to understand or modify the code, first thing to do is to read the article.
Here are the details specific to the houdini implementation.

NOTE: the houdini name of any the node created by this code contains "FS" so you can find them
  more easily in Houdini.

A set of sources is encoded as a geometry detail:
  * Detail:
      -Attibutes:
         buffer_size (int): (aperiodic version) size of the buffer recording past amplitude
	                          for each source (number of float, as amplitudes are complexes
				  buffer_size/2 amplitudes are recorded)
			    (periodic version)	should be 2.  
	 damping (float)
  *Primitive: subset of source of same wavelength
      -Attibutes:
         wavelengths (float)
	 ampli_step (int): amplitudes of the sources updated every <ampli_step> time step
  *Point: one source of one wavelength
      -Attibutes:
        P (UT_Vector3): position, (x, z) should be the position of the surface, y=0
	ampli (FloatTuple of size <buffer_size>): complex amplitudes of the source
	           ampli[2n], ampli[2n+1]: real and imaginay part of the amplitude after n time step
(Note that the wavelength of one source point is defined be their corresponding primitive)

Representation of the surface:
We use a regular or projected grid, but any planar surface mesh should work.
For the periodic method, each node of the grid store a spectrum (a complex amplitude for each 
  wavelength).


SOP_Create_Source/SOP_Create_Sources:
create one source/a set of sources (from a set of point) that can be used as input.
  Parameters:
     -Position (X, Y, Z): position of the source (X, Z: coordintate on the water surface, Y=0)
     -Amplitude
     -Phase
     -Minimum/maximum wavelength, wavelength multiplicative step: used to compute the range of wl
     -Type (not used yet, keep at 0)
     -Size of the buffer: size of the buffer recording past amplitude
     -Damping



SOP_Circle/Square/TextureObstacle:
Create set of point sources along an offset surface of the obstacle.
Range of wavelength (and time step per wl), and is copied from input geometry, as well as
  the detail attibute.
Create on primitve and subset of sources for each wl.
Spacing depends on the wavelength and the parameter "density".
I use this node also to create a set of point sampling the border of the obstacle (offset=0)
  for the boundary conditions.
Note: the density of the boundary points should be at least twice the density of the
  sources.
Note 2: for the TextureObstacle the density is also limited by the resolution of the
  texture (cannot have more than one source per pixel).
Note 3: for the aperiodic version, do not forget to check the "interactive sources" box in
  the parameter of the node creating the sources of the obstacle.

SOP_Solve_FS/SOP_Solve_FS_inter:
Compute amplitude of the obstacle sources (input 0) such that the boundary condition are
  respected (as well as possible, least square) at the boundary points (input 1) given the
  incoming waves (input 2).

SOP_Deform_Surface/SOP_Deform_Surface_inter:
Compute height at each point of the grid (input 0) by summing all the sources.
(each other input is a set of sources)

SOP_Merge_Sources:
Take any number (at least one) of set of sources and merge them together.
For now, we assume tht all the sets have the same parameter (range of wavelength, 
  buffer_size etc...).

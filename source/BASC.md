# BASC pipeline

## Overview

This tutorial describes how to apply a bootstrap analysis of stable clusters (BASC) on a group of fMRI datasets acquired on multiple subjects. The bootstrap analysis of stable clusters (BASC) is a pipeline that builds brain parcellations (clusters) based on the similarity of individual fMRI time series. The BASC implements some boostrap replications of the cluster analysis as well as a consensus clustering approach to capture stable clusters at the individual and group levels. The pipeline also includes an automated method (called MSTEPS) to identify critical numbers of clusters, that summarize accurately a whole hierarchy of decomposition into brain networks. There is a set of [slides](http://files.figshare.com/2030483/basc_pipeline_niak.pdf) describing the pipeline.  

The command to run the pipeline in a Matlab/Octave session is:
```matlab
niak_pipeline_stability_rest(files_in,opt)
```
where ''files_in'' is a structure describing how the dataset is organized, and ''opt'' is a structure describing the options of the pipeline. The code of [niak_template_stability_rest](https://raw.githubusercontent.com/SIMEXP/niak/master/template/niak_template_stability_rest.m) would be a good starting point to write your own script.

## Input files

The inputs of the pipelines are:

 * Preprocessed fMRI datasets.  
 * A (3D) binary brain mask. The BASC will be applied only to the voxels within the mask.
 * A mask of brain areas. The region growing is applied on the concatenated time series of all subjects in each area to limit the memory demand.

### Preprocessed fMRI data

If the fMRI datasets have been preprocessed using NIAK, setting up the input files is very easy. Just grab the results of the preprocessing with the following command:

```matlab
% The minimum number of volumes for an fMRI dataset to be included.
% This option is useful when scrubbing is used, and the resulting time series may be too short.
opt_g.min_nb_vol = 100;
% Specify to the grabber to prepare the files for the region growing pipeline
% opt_g.type_files = 'roi';
files_in = niak_grab_fmri_preprocess('/home/pbellec/demo_niak_preproc/',opt_g);
```

More options for the grabber are available. See ''help niak_grab_fmri_preprocess''. If NIAK was not used to prepocess the data, all inputs have to be manually specified in the script. The first field ''fmri'' directly lists all of the preprocessed fMRI datasets, organized by subject, session and runs. Example:

```matlab
 files_in.data.subj1.sess1.rest = '/data/fmri_subject1_session1_rest.nii.gz';
 files_in.data.subj2.sess1.rest = '/data/fmri_subject2_session1_rest.nii.gz';
```

**WARNING**: Octave and Matlab impose some restrictions on the labels used for subject, session and run. In order to avoid any issue, please do not use long labels (say less than 8 characters for subject, and ideally 4 characters or less for session/run). Also avoid to use any special character, including `.` `+`, `-` or `_`. None of these restrictions apply on the naming convention of the files, just to the labels that are used to build the structure files_in in Matlab/Octave.

### Analysis mask

The ''mask'' field is the name of a 3D binary volume serving as a mask for the analysis. If the results of the fMRI preprocessing pipeline have been grabbed, the mask is automatically set to the the group mask generated in the quality control (i.e. voxels that fall in the brain for more than 50% of subjects). It can be a mask of the brain common to all subjects, or a mask of a specific brain area, e.g. the thalami. It is important to make sure that this segmentation is in the same space and resolution as the fMRI datasets. If not, use SPM/SPM or MINCRESAMPLE to resample the mask at the correct resolution.
```matlab
 files_in.mask = '/home/pbellec/demo_niak_preproc/quality_control/group_coregistration/func_mask_group_stereonl.nii.gz';
```

### Brain areas

Finally, the ''areas'' field is the name of a volume of integer values, describing some brain areas that are used to save memory space in the region-growing algorithm. By default, NIAK will use a modified AAL template that covers the full cerebellum, and is adapted to the MNI152 non-lilnear symmetric template. Different brain areas are treated independently at this step of the analysis. It is important to make sure that this segmentation is in the same space and resolution as the fMRI datasets. If not, use SPM or mincresample to resample the AAL segmentation at the correct resolution.
```matlab
files_in.areas = '/data/roi_aal.nii.gz';
```

## Options

The different options are passed through fields in the structure ''opt''.

The first option is the name of the folder where the results will be stored. Note that this folder does not need to be created before hand. Example:

```matlab
  opt.folder_out = '/data/basc/'; % Where to store the results
```

The second option is the size of the regions in the region growing algorithm, when they are mature (i.e. they stop growing). The parameter is set in mm3, if the size of voxels is specified in mm in the header (that is pretty much always the case). A threshold of 1000 mm3 will give about 1000 regions on the grey matter.

```matlab
opt.region_growing.thre_size = 1000;
```

The following list will set the grid of scales (number of clusters) explored to identify stable clusters. The following grid will search from 10 to 500 clusters, with more granularity at low scales:

```matlab
opt.grid_scales = [10:10:100 120:20:200 240:40:500]';
```

The following scales will be used to generate the maps of brain clusters and stability. In this example the same number of clusters are used at the individual (first column), group (second column) and consensus (third and last colum) levels. It is also possible to leave this parameter empty, and then manually copy the results of the MSTEPS selection before restarting the pipeline.

```matlab
opt.scales_maps = repmat(opt.grid_scales,[1 3]);
```

The number of bootstrap samples at the individual level (100: the CI on indidividual stability is +/-0.1):

```matlab
opt.stability_tseries.nb_samps = 100;
```

The number of bootstrap samples at the group level (500: the CI on group stability is +/-0.05):

```matlab
opt.stability_group.nb_samps = 500;
```

The following flag turns on/off the generation of maps/time series at the individual level:

```matlab
opt.flag_ind = false;
```

The following flag turns on/off the generation of maps/time series at the mixed level (group-level networks mixed with individual stability matrices):

```matlab
opt.flag_mixed = false;
```

The following flag turns on/off the generation of maps/time series at the group level:

```matlab
opt.flag_group = true;
```

## Outputs

A number a subfolders are created in the ''opt.folder_out'' directory. In the following, EXT will denote the extension associated with the file type of the functional images, e.g. ''.nii'' or ''.nii.gz'' for nifti. An exhaustive description of the outputs follows. Most of them may not be of interest. The main results have been highlighted with a 8-o. The results that are generated only if some scales are listed in the second pass are indicated by a `*`.

The `areas` subfolder contains the time series and mask of the areas used to perform region growing:

 * **`brain_areas.EXT`**: a 3D volume with integer values. The `I`th area is filled with `I`s.
 * **`brain_areas_neig.mat`**: a .mat file which contains a bunch of variable. `ind_I` is a vector with the linear index of every voxels in brain area `I`. `neig_I` is a cell of vector. The `k`th entry is the list of the spatial neighbors of the voxel `k` in the `I` region. Note that all indices used here refer to entries in the list `ind_I`, which means that the `k`th entry correspond to the voxel `ind_I(k)`.
 * **`part_areas_I.mat`**: a mat file with one vector `part`. `part(k)` is the number of the region voxel `k` of area `I` belongs to after region growing.

The **`logs`** folder keeps track of all the execution of the pipeline (see below the section on pipeline management). This folder needs to be left intact at all time. It contains all the logs of the pipeline execution, but no results directly relevant to BASC.

The **`rois`** subfolder contains the ROIs generated through region-growing along with the associated time series for all runs and subjects:

* **`brain_rois.EXT`**: a 3D volume with integer values. The `I`th ROI is filled with `I`s.
* **`tseries_<subject>_<run>.mat`**: a .mat file with one array `tseries`. The `I`th column of `tseries` is the time series associated with region `I` for subject `<subject>` and run `<run>`

The **`multiscale`** folder contains information about the exploration of stable clusters for different number of clusters.

* **`silhouette.mat`**: a .mat file with the following variables&nbsp;:
* **`scale_ind`**: vector, `scales_ind(k)` is the number of individual clusters for experiment `k`.
* **`scale_group`**: vector, `scales_group(k)` is the number of group clusters for experiment `k`.
* **`sil`**: array, `sil(m,k)` is the stability contrast criterion for `scale_ind(k)` clusters at the individual level, `scale_group(k)` clusters at the group level and `m` final clusters.
* **`sil_max`**: vector, `sil_max(m)` is the maximal stability contrast achieved for `m` final clusters, and the individual/group number of clusters in a neighbourhood of `m`.
* **`scales_ind_max`**: vector, `scales_ind_max(m)` is the number of individual clusters that achives `sil_max(m)` for `m` final clusters.
* **`scales_group_max`**: vector, `scales_group_max(m)` is the number of group clusters that achives `sil_max(m)` for `m` final clusters.
* **`table_multiscale.dat`**: a text file with a table. For all scales (meaning a set of individual, group and final number of clusters) investigated in the second pass, the table recapitulates the relative overlap of the clusters identified at different scales.
* **`figure_sigma_max.pdf`**: a pdf with the locally maximal stability contrast criterion as a function of the final number of clusters, along with the associated number of individual clusters and group clusters.
* **`table_sigma_max.dat`**: a text file recapitulating the local maxima of the stability contrast criterion, along with the corresponding number of clusters at the individual, group and final levels. These parameters can be used in the second pass analysis when re-running the pipeline.

The folder **`stability_ind`** contains the results of BASC at the individual level. The subfolders `sci<k>` contains the results for `k` individual clusters:

* **`stability_ind_<subject>_sci<k>_pass1.mat`**: a .mat file with a unique variable (vector) `stab`, which is the vectorized form of the individual stability matrix for subject <subject> and <k> individual clusters, for the first pass. Use `niak_vec2mat` to get the matrix form.
* **`stability_ind_<subject>_sci<k>_pass1.mat`**: a .mat file with a unique variable (vector) ''stab'', which is the vectorized form of the individual stability matrix for subject &lt;subject&gt; and &lt;k&gt; individual clusters, for the second pass. Use ''niak_vec2mat'' to get the matrix form.

The folder **`stability_group`** contains the results of BASC at the group level. The subfolders `sci<k>_scg<l>` contains the results for `<k>` individual clusters and `<l>` group clusters:

* **`brain_networks_sci<k>_scg<l>_scf<m>.EXT`**: a 3D map where the ''I''th group stable cluster is filled with ''I''s. A threshold is in addition applied on the regions of the cluster such as the average stability is greater than 0.5. This can change the total number of stable clusters.
* **`brain_partition_sci<k>_scg<l>_scf<m>.EXT`**: a 3D map where the ''I''th group stable cluster is filled with ''I''s.
* **`map_networks_group_sci<k>_scg<l>_scf<m>.EXT`**: a 3D map where the value in a region is the associated average group stability regarding the group stable network this region belongs to.
* **`map_networks_avg_ind_sci<k>_scg<l>_scf<m>.EXT`**: a 3D map where the value in a region is the associated average individual stability regarding the group stable network this region belongs to.
* **`map_networks_&lt;subejct&gt;_sci<k>_scg<l>_scf<m>.EXT`**: a 3D map where the value in a region is the associated individual stability for subject ''&lt;subject&gt;'' regarding the group stable network this region belongs to.
* **`partition_networks_sci<k>_scg<l>_scf<m>.mat`**: a .mat files with the following variables&nbsp;:
  * **`part`** (array) ''part(1,:)'' is the partition of regions into stable group clusters. ''part(1,i)'' is the number of the cluster associated with region ''i''. ''part(2,:)'' is the same, but after a threshold of 0.5 for the average group stability was applied on the regions.
  * **`order`** (vector) is an ordering of the region deduced of the HAC on the group-level stability matrix. It is used to represent all stability matrices.
  * **`hier`** (array) is a representation of the final HAC on the group-level stability matrix. See `niak_hierarchical clustering` for more infos.
  * **`stability_group_sci<k>_scg<l>_pass1.mat`**: a .mat file with a number of variables:
  * **`stab_ind`** (vector) the vectorized form of the average individual stability matrix for `k` individual clusters and `l` group clusters, for the first pass. Use `niak_vec2mat` to get the matrix form.
  * **`stab_group`** (vector) the vectorized form of the group stability matrix for `k` individual clusters and `l` group clusters, for the first pass. Use `niak_vec2mat` to get the matrix form.
  * **`stab_plugin`** (vector) the vectorized form of the adjacency matrix resulting of HAC applied on the average individual stability matrix. It is the plugin estimate of the group adjacency matrix for &lt;k&gt; individual clusters and `l` group clusters, for the first pass. Use `niak_vec2mat` to get the matrix form.
  * **`stability_group_sci<k>_<t>_pass2.mat`**: a .mat file with a number of variables:
  * **`stab_ind`** (vector) the vectorized form of the average individual stability matrix for `k` individual clusters and `l` group clusters, for the second pass. Use `niak_vec2mat` to get the matrix form.
  * **`stab_group`** (vector) the vectorized form of the group stability matrix for `k` individual clusters and `l` group clusters, for the second pass. Use `niak_vec2mat` to get the matrix form.
  * **`stab_plugin`** (vector) the vectorized form of the adjacency matrix resulting of HAC applied on the average individual stability matrix. It is the plugin estimate of the group adjacency matrix for `k` individual clusters and `l` group clusters, for the second pass. Use `niak_vec2mat` to get the matrix form.
* **`table_clusters_group_sci<k>_scg<l>_scf<m>.dat`**: a text file containing a table that recapitulates the local peaks of stability within each cluster.
* **`table_networks_group_sci<k>_scg<l>_scf<m>.dat`**: a text file containing a table that recapitulates the relative overlap of each cluster with Brodmann and AAL areas.
* **`tseries_networks_sci<k>_scg<l>_scf<m>_<subject>_<run>.mat`**: a .mat file with the following variables:
  * **`tseries`**: the Ith column is the weighted average time series associated with cluster I, for `k` individual clusters, `l` group clusters, `m` final clusters. In the computation of the average, the contribution of each ROI to the mean is weighted by its stability.
  * **`tseries_mean`**: the Ith column is the mean time series associated with cluster I, for `k` individual clusters, `l` group clusters, `m` final clusters.
  * **`tseries_std`**: the Ith column is the standard deviation of the mean of the time series associated with cluster I, for `<k>` individual clusters, `<l>` group clusters, `<m>` final clusters.

## Publication guidelines

Here is a short description of the BASC pipeline that can be adapted in a publication. You are encouraged to include the script that was used to run the pipeline as supplementary material of the article.

The fMRI data was first reduced to N time*space arrays using a region-growing algorithm, where N is the number of subjects and the spatial dimension is essentially selected by the user through a parameter which is the size of the regions when the region growing stops. Typically, a threshold of 1000 mm3 will result into around 1000 regions for a whole-brain analysis restricted to grey matter. The regions are built to maximize the average correlation between time series within each region for all subjects.

The basic principle of BASC is to repeat a clustering operation a large number of times, and compute the frequency at which each pair of regions fall in the same cluster. This stability matrix is then itself used in a clustering procedure, which derives so-called stable clusters (stable clusters are composed of regions with a high probability of falling in the same clusters, hence the name). That principle is first applied on individual fMRI time series, and replications of clustering are derived by applying a circular block bootstrap to fMRI time series. A stability matrix is derived for each subject. The average individual stability matrix is then computed and a hierarchical clustering is applied on it. This allows to define group-level clusters which maximize the average probability of being clustered at the individual level. By bootstrapping the subjects of the population, the group-level clustering itself can be subject to BASC, and a group-level stability matrix is derived. A hierarchical clustering is
finally applied on the group stability matrix to define stable group clusters. For each brain region, the average stability of that region with every other regions in the cluster of the seed can be derived. This can be done with the individual, average individual or group stability matrices, hence generating stability maps at all levels of analysis for every stable group cluster.

 There are three main clustering parameters: the number of clusters at the individual, group and final levels (K, L and M respectively). The contrast of the group stability within and between stable group clusters is derived as a measure of the overall clustering quality. In a first pass, the space of all possible (K,L,M) is explored with a small number of bootstrap samples to save time. For each M, the maximal contrast for K and L in a neighbourhood of M is derived. It is then possible in a second pass to generate stability matrices with a high number of bootstrap samples for the number of clusters which are local maxima of the contrast.

Reference:

 P. Bellec;P. Rosa-Neto; O. C. Lyttelton; H. Benali; A. Evans, Multi-level bootstrap analysis of stable clusters in resting-state fMRI. NeuroImage, Volume 51, Issue 3, 1 July 2010, Pages 1126-1139

# SAR_amp_change
code related to calculating amplitude change products from SAR data

gee_amp_change_detection is meant to be run in Google Earth Engine
You will need a GEE account
copy and paste the text of gee_amp_change_detection into a new script in GEE and run


############### Changes I want to make ################
- have it handle merging SAR frames 
- make it easier to toggle what calculations to do/plot
- make decent quality figures to export, not just tifs
- add in options to find and plot optical before/after imagery

################### Background Info ###################
- SAR backscatter data is one way that people use to detect change between two acquisitions
- Google Earth Engine provides a dataset of SAR backscatter converted to dB 
- other common ways of displaying SAR backscatter data are intensity (I) and amplitude (A)
	dB = 10*log_10(I)
	A = sqrt(I)
- Sentinel-1 (the satellite constellation this script is for) has co- and cross-polarized data
	VV & VH
	VH (cross-polarized) data can be more helpful in vegetated areas because it is sensitive to volume scattering
	but it is typically more noisy than VV data

################ Script Functionality ################
- this script takes two scenes: one before and one after an event of interest
- it calculates A and I for each scene and polarization
- it calculates the following:
	log difference of intensity
		log_diff_I = log_10(avgBefore/avgAfter) where avgBefore and avgAfter are the averages of VV + VH intensity

	log ratio of intensity (VV & VH) - only uses one polarization at a time (from Jung & Yun, 2020)
		log_ratio_I = 10*log(Ibefore/Iafter)

	dB difference (VV & VH)
		diff_dB = dBafter-dBbefore

	ratio of dB (VV & VH)
		ratio_dB = dBbefore/dBafter

	RGB image with R = dBafter, G = dBbefore, B = ratio_dB (from Gosling et al., 2025)

	correlation using dB
		using the GEE reduceNeighborhood function

	
################# References ##################
Jung, J., & Yun, S.-H. (2020). Evaluation of Coherent and Incoherent Landslide Detection Methods Based on Synthetic Aperture Radar for Rapid Response: A Case Study for the 2018 Hokkaido Landslides. Remote Sensing, 12(2), 265. https://doi.org/10.3390/rs12020265
Gosling, J., Dualeh, E. W., & Biggs, J. (2025, January 10). Analysis and automatic detection of lava flows using SAR backscatter applied to the 2017 eruption of Erta ’Ale Volcano, Ethiopia. In Review. https://doi.org/10.21203/rs.3.rs-5003481/v1

###############################################
script written by Elizabeth Eiden, updated 6/11/25

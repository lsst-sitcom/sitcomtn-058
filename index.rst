:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.


Abstract
========

A catalog of clean surrounding stars is required to perform focus and alignment with the AuxTel observation. It is anticipated that rebuilding star catalog is required when new criteria should be applied on the selection of stars. This technote is to describe the notebook that creates new catalog based on queries from Tycho-2 and HD catalog. This notebook can be found on this github repo `https://github.com/lsst-sitcom/sitcomtn-058/tree/main/_static <https://github.com/lsst-sitcom/sitcomtn-058/tree/main/_static>`__. 


Generating a star catalog for the Auxtel observation
====================================================


The following description is about the Juypter notebook to generate the star catalog with required selection criteria. 
See also details about `focusing, center, and absorb pointing offsets at AuxTel <https://obs-ops.lsst.io/Nighttime-Operations/Auxiliary-Telescope/AT-On-Sky/Focus-center-absorbPointingOffsets.html>`__.



Create a notebook with the Tycho-2 catalogue
--------------------------------------------
First, a list of stars from Vizier query of  
`Tycho-2 catalogue <http://vizier.cds.unistra.fr/viz-bin/VizieR-3?-source=I/259/tyc2>`__ :cite:`2000A&A...355L..27H`. 
The range of :abbr:`RA (Right ascension)`, :abbr:`Decl. (Declination)`, and :math:`{V_{T}}`.

- ``file``: ``string``, the json file name of Tycho-2 query file. 

- ``RAmdeg``: ``string``, mean Right Asc, ICRS, epoch=J2000 with logical operator

- ``Demdeg``: ``string``, mean Decl, ICRS, at epoch=J2000 with logical operator

- ``VTmag``: ``string``, Tycho-2 VT magnitude with logical operator

The default selection criteria is all stars with ``DEmdeg = '<10.0'`` and ``VTmag = '<10'`` and the name of output file is ``file = './cwfs_tycho2_stars.pd'``.

Cutting Selection Criteria
--------------------------

If this is the first time using this Jupyter notebook, Tycho-2 catalog with constraints will be quried from Vizier and save the list of all stars with constraints into json file ``file = './cwfs_tycho2_stars.pd'``. If the file already exists in local, it reads the existing file.

.. code-block:: py
   
    if os.path.exists(file):
        print('Read existing Vizier Catalog in Local'+file)
        tycho_star_all = pd.read_json(file)
    else :
        print('Read Vizier Catalog I/259/tyc2 ')
        target_stars = Vizier.query_constraints(catalog='I/259/tyc2', DEmdeg=DEmdeg, VTmag=VTmag)[0]
        result = target_stars.to_pandas().to_json(file)

In order to execlude targets contaminated from bright nearby star, following criteria are further applied to the Tycho-2 catalog:

- Include stars with 6 < V < 8 

- Elimiate stars with

  - very bright (V < 10 mag) nearby stars within 1 arcmin. radius 
  - bright (V < 6 mag) within 7 arcmin radius
  - extremely nearby stars (V < 3mag) within 20 arcmin radius  

- Exclude stars with higher proper motion (> 500 mas/yr) 

These constraints can be also changed with following parameters:

- ``mag_upper``, ``mag_lower``: upper and lower V magnitude limit of the CWFS stars. The default values are :math:`V_{upper}` = 6 mag, :math:`V_{lower}` = 8 mag. 

- ``pm_cut``: upper limit of proper motion (:math:`\sqrt{(RA_{pm}\cdot{cos(Decl))}^{2}+{{Decl_{pm}}^{2}}}`). The default value is +-500mas/yr.

- ``mag_cutN``, ``area_cutN`` : magnitude cut and area for Nth criteria.

.. code-block:: py

    mag_upper = 6 ; mag_lower = 8; pm_cut =500 #pm_cut for net pm (sqrt(pmRA*cosDec^2.0+pmDEC^2.0))
    area_cut1 = 1; area_cut2 = 7; area_cut3 = 20 #arcmin.
    mag_cut1 = 10; mag_cut2 = 6; mag_cut3= 3 #Vmag

    tycho_star_all['net_pm'] = 0.5*(((tycho_star_all['pmRA'])**2.0 + (tycho_star_all['pmDE'])**2.0)**0.5)
    tycho_star_cut = tycho_star_all[(tycho_star_all['VTmag'] > mag_upper) & (tycho_star_all['VTmag'] < mag_lower) & (tycho_star_all['net_pm'] < pm_cut)]

    def sep(RA1,RA2,Dec1,Dec2):
    sep = ((((RA1-RA2)*np.cos(Dec1))**(2.0))+((Dec1-Dec2)**2.0))**(0.5)
    return sep

    tycho_star_cut_criteria = pd.DataFrame.copy(tycho_star_cut)

    for i in tycho_star_cut.index:
        within_1arcmin = (sep(tycho_star_cut["RA_ICRS_"][i],tycho_star_all["RA_ICRS_"],tycho_star_cut["DE_ICRS_"][i],tycho_star_all["DE_ICRS_"]) < area_cut1/60.0) & (tycho_star_all["VTmag"] < mag_cut1)
        within_7arcmin = (sep(tycho_star_cut["RA_ICRS_"][i],tycho_star_all["RA_ICRS_"],tycho_star_cut["DE_ICRS_"][i],tycho_star_all["DE_ICRS_"]) < area_cut2/60.0) & (tycho_star_all["VTmag"] < mag_cut2)
        within_20arcmin = (sep(tycho_star_cut["RA_ICRS_"][i],tycho_star_all["RA_ICRS_"],tycho_star_cut["DE_ICRS_"][i],tycho_star_all["DE_ICRS_"]) < area_cut3/60.0)& (tycho_star_all["VTmag"] < mag_cut3)
        if np.count_nonzero(within_1arcmin) > 1 or np.count_nonzero(within_7arcmin) > 0 or np.count_nonzero(within_20arcmin) > 0:
            tycho_star_cut_criteria.drop([i], inplace=True)
.. table::

    .. csv-table:: Selected Tycho-2 Stars 
        :header: , TYC1,TYC2,TYC3,pmRA,pmDE,BTmag,VTmag,HIP,RA_ICRS,DE_ICRS,net_pm
        :width: 20

        1,4663,45,1,45.900002,-53.400002,7.695,6.420,417.0,1.265827,-0.502912,35.207848
        6,4663,160,1,62.200001,-15.300000,8.819,7.399,14.0,0.048280,-0.360421,32.027059
        7,4663,363,1,7.600000,-4.200000,7.972,6.349,664.0,2.050386,-2.447699,4.341659
        25,4663,1285,1,40.200001,-4.700000,8.318,7.164,700.0,2.179543,-2.222573,20.236910
        36,4664,560,1,-9.100000,35.400002,8.027,7.051,1587.0,4.961996,-2.478966,18.275462
        ...,...,...,...,...,...,...,...,...,...,...,...
        193594,1170,1,1,-49.500000,-182.199997,8.179,7.538,117236.0,356.576572,9.785502,94.402183
        193598,1170,247,1,16.799999,-21.100000,9.190,7.819,116978.0,355.704470,9.885516,13.485640
        193605,1170,883,1,31.000000,-26.500000,7.024,6.684,117394.0,357.050458,8.245658,20.391481
        193626,1171,425,1,15.600000,-8.800000,6.827,6.818,117962.0,358.907095,8.223298,8.955445
        193636,1171,1243,1,10.200000,0.300000,9.586,7.936,117912.0,358.750162,8.389472,5.102205


Match Tycho-2 and HD catalogs
-----------------------------
Now the star list is matched to `HD identifications for Tycho-2 stars <http://vizier.cfa.harvard.edu/viz-bin/VizieR-3?-source=IV/25/tyc2_hd>`__ :cite:`2002A&A...386..709F`.


.. code-block:: py

    HD_stars_all = Vizier.query_constraints(catalog='IV/25/tyc2_hd')[0]
    HD_star_match= Table(HD_stars_all[0:0])
    for i in tycho_star_cut_criteria.index: 
        condition = (HD_stars_all["TYC1"]==tycho_star_cut_criteria["TYC1"][i]) & \
        (HD_stars_all["TYC2"]== tycho_star_cut_criteria['TYC2'][i]) & \
        (HD_stars_all["TYC3"]==tycho_star_cut_criteria['TYC3'][i])
        if np.count_nonzero(condition) == 1:
            table = HD_stars_all[condition]
            HD_star_match.add_row(table[:][0])

Query Simbad data for star sample
---------------------------------
Then, query simbad data for each selected CWFS star. 

- The default VOTable fields: ``MAIN_ID``, ``RA``, ``DEC``, ``RA_PREC``, ``DEC_PREC``, ``COO_ERR_MAJA``, ``COO_ERR_MINA``, ``COO_ERR_ANGLE``, ``COO_QUAL``, ``COO_WAVELENGTH``, ``COO_BIBCODE``, ``SCRIPT_NUMBER_ID``

- Add ``flux_name(V)``, ``flux(V)``, ``flux_error(V)``, ``flux_system(V)``, ``flux_bibcode(V)``, ``flux_qual(V)``, ``flux_univ(V)``  VOTable fields. 

- If it is required to add another Simbad VOTable fields, check ``Simbad.list_votable_fields()`` and fields using ``add_votable_fields()``.


.. code-block:: py

    customSimbad = Simbad()
    customSimbad.add_votable_fields('flux_name(V)', 'flux(V)', 'flux_error(V)', 'flux_system(V)','flux_bibcode(V)', 'flux_qual(V)', 'flux_unit(V)')

    final = customSimbad.query_object('HD '+str(HD_star_match["HD"][0]))

    for i in range(1,len(HD_star_match["HD"])):
        result_table = customSimbad.query_object('HD '+str(HD_star_match["HD"][i]))
        final.add_row(result_table[:][0])

.. table::
  
    .. csv-table:: Final Output from Simbad Query
        :header: MAIN_ID,RA,DEC,RA_PREC,DEC_PREC,COO_ERR_MAJA,COO_ERR_MINA,COO_ERR_ANGLE,COO_QUAL,COO_WAVELENGTH,COO_BIBCODE,FILTER_NAME_V,FLUX_V,FLUX_ERROR_V,FLUX_SYSTEM_V,FLUX_BIBCODE_V,FLUX_QUAL_V,FLUX_UNIT_V,SCRIPT_NUMBER_ID
       
        HD 6,00 05 03.8227,-00 30 10.928,14,14,0.037,0.023,90,A,O,2020yCat.1350....0G,V,6.298,0.010,Vega,2000A&A...355L..27H,D,V,1
        HD 224726,00 00 11.6217,-00 21 37.608,14,14,0.021,0.017,90,A,O,2020yCat.1350....0G,V,7.27,--,Vega,,E,V,1
        * 5 Cet,00 08 12.0955,-02 26 51.740,14,14,0.059,0.036,90,A,O,2020yCat.1350....0G,V,6.22,--,Vega,,E,V,1
        HD 406,00 08 43.1091,-02 13 21.296,14,14,0.036,0.030,90,A,O,2020yCat.1350....0G,V,7.05,0.010,Vega,2000A&A...355L..27H,D,V,1
        HD 1567,00 19 50.8746,-02 28 43.990,14,14,0.028,0.015,90,A,O,2020yCat.1350....0G,V,6.96,0.010,Vega,2000A&A...355L..27H,D,V,1
        ...,...,...,...,...,...,...,..,...,...,...,...,...,...,,...,...,...
        HD 1421,00 18 18.5194,-02 00 53.291,14,14,0.021,0.014,90,A,O,2020yCat.1350....0G,V,7.18,--,Vega,,E,V,1
        HD 999,00 14 24.4641,-02 11 52.802,14,14,0.022,0.018,90,A,O,2020yCat.1350....0G,V,7.18,0.010,Vega,2000A&A...355L..27H,D,V,1
        HD 820,00 12 40.3372,-01 13 37.885,14,14,0.019,0.015,90,A,O,2020yCat.1350....0G,V,7.2,0.010,Vega,2000A&A...355L..27H,D,V,1
        HD 1369,00 17 48.3722,-01 51 45.858,14,14,0.068,0.055,90,A,O,2020yCat.1350....0G,V,7.1,0.010,Vega,2000A&A...355L..27H,D,V,1
        HD 2023,00 24 29.6495,-02 13 08.626,14,14,0.027,0.018,90,A,O,2020yCat.1350....0G,V,6.067,0.010,Vega,2000A&A...355L..27H,D,V,1



Exclude individual stars manually (optional)
--------------------------------------------
This section is to exclude the stars from the list manually. Put HD of the stars on the ``Remove_main_id`` parameter. Even if there are any stars not included in the final table, it is fine to run. 

.. code-block:: py
    
    Remove_main_id = ["HD22746","HD452"] #HD NNNNNN 
    p= re.compile("\d*\.?\d+")
    customSimbad = Simbad()
    for i in range(len(Remove_main_id)):
        number = p.findall(Remove_main_id[i])[0]
        Remove_main_id_simbad= customSimbad.query_region('HD '+str(number))["MAIN_ID"]
        Remove = (final["MAIN_ID"] == Remove_main_id_simbad[0])
        if np.count_nonzero(Remove) !=0 :
            remove_index = [i for i, x in enumerate(Remove) if x]
            final.remove_row(remove_index[0])
            print(Remove_main_id_simbad[0]+' is now removed from the final catalog')


Save the catalog on the output file
-----------------------------------
As a final step, the queried table is saved into json file. The default name for output is :file:`HD_cwfs_stars.pd`. The file name can changed with ``file_name_final`` variable. 

.. code-block:: py
   
    file_name_final = 'HD_cwfs_stars.pd' #file_name
    result = final.to_pandas().to_json(file_name_final)
    print('List of Stars was exported to '+file_name_final)

Plot for the distribution of the Stars on Sky
=============================================
To check wheather selected CWFS stars are homogeneously distributed on the southern sphere, equatorial coordinates RA, Dec of each starsare plotted on Mollweide projection.

.. image:: /_static/dist_stars.png




Appendix. Check the Field of the Individual Star
================================================
When checking the FOV of the individual star, you can check it manually.<br>
The default FOV of the image are 6.7' x 6.7'. 

.. code-block:: py
   :name: finding-chart-generator 
    
   star_name_img_query = "HD 2527"
   FOV=6.7*1/60.0 #6.7 x 6.7 arcminutes for AuxTel

   from astroquery.skyview import SkyView
    import numpy as np
    survey_name = ["DSS2 Blue", "DSS2 Red", "DSS2 IR"]
    img = SkyView.get_images(star_name_img_query,survey=survey_name,\
                         height=FOV*u.degree,width=FOV*u.degree,coordinates='J2000',grid=True,gridlabels=True)

    ncol=len(survey_name)
    fig,ax = plt.subplots(ncols=ncol,figsize=(24,8))
    
    for i in range(ncol):
        plot = ax[i].imshow(img[i][0].data,vmax=np.max(img[i][0].data)*.95,\
                        vmin=np.max(img[i][0].data)*.25, aspect='equal')
    ax[i].set_title(str(survey_name[i]), fontsize=15)
    fig.gca().invert_yaxis()

    print(star_name_img_query, 'FOV = '+str((FOV*60))+'\"'+'x '+str((FOV*60))+'\"')




.. image:: /_static/finding_chart.png






.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations

.. rubric:: References
.. bibliography:: local.bib 
    :style: lsst_aa

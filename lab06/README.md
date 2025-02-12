Lab 06 - Regular Expressions and Web Scraping
================

# Learning goals

- Use a real world API to make queries and process the data.
- Use regular expressions to parse the information.
- Practice your GitHub skills.

# Lab description

In this lab, we will be working with the [NCBI
API](https://www.ncbi.nlm.nih.gov/home/develop/api/) to make queries and
extract information using XML and regular expressions. For this lab, we
will be using the `httr`, `xml2`, and `stringr` R packages.

This markdown document should be rendered using `github_document`
document ONLY and pushed to your *JSC370-labs* repository in
`lab06/README.md`.

## Question 1: How many sars-cov-2 papers?

Build an automatic counter of sars-cov-2 papers using PubMed. You will
need to apply XPath as we did during the lecture to extract the number
of results returned by PubMed in the following web address:

    https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2

Complete the lines of code:

``` r
# Downloading the website
website <- xml2::read_html("https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2")

# Finding the counts
counts <- xml2::xml_find_first(website, "/html/body/main/div[9]/div[2]/div[2]/div[1]/div[1]/h3/span")

# Turning it into text
counts <- as.character(counts)

# Extracting the data using regex
stringr::str_extract(counts, "[0-9,]+")
```

    ## [1] "247,751"

- How many sars-cov-2 papers are there?

247,751 papers

Don’t forget to commit your work!

## Question 2: Academic publications on sars-cov-2 related to Canada

Use the function `httr::GET()` to make the following query:

1.  Baseline URL:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi>

2.  Query parameters:

    - db: pubmed
    - term: covid19 toronto
    - retmax: 1000

The parameters passed to the query are documented
[here](https://www.ncbi.nlm.nih.gov/books/NBK25499/).

``` r
library(httr)
query_ids <- GET(
  url   = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi",
  query = list(
    db = "pubmed",
    term = "covid19 toronto", 
    retmax = 300
  )
)

# Extracting the content of the response of GET
ids <- httr::content(query_ids)
```

The query will return an XML object, we can turn it into a character
list to analyze the text directly with `as.character()`. Another way of
processing the data could be using lists with the function
`xml2::as_list()`. We will skip the latter for now.

Take a look at the data, and continue with the next question (don’t
forget to commit and push your results to your GitHub repo!).

## Question 3: Get details about the articles

The Ids are wrapped around text in the following way:
`<Id>... id number ...</Id>`. we can use a regular expression that
extract that information. Fill out the following lines of code:

``` r
# Turn the result into a character vector
ids <- as.character(ids)

# Find all the ids 
ids <- stringr::str_extract_all(ids, "<Id>[0-9]+</Id>\n")[[1]]

# Remove all the leading and trailing <Id> </Id>. Make use of "|"
ids <- stringr::str_remove_all(ids, "<Id>|</Id>\n")
```

With the ids in hand, we can now try to get the abstracts of the papers.
As before, we will need to coerce the contents (results) to a list
using:

1.  Baseline url:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi>

2.  Query parameters:

    - db: pubmed
    - id: A character with all the ids separated by comma, e.g.,
      “1232131,546464,13131”
    - retmax: 1000
    - rettype: abstract

**Pro-tip**: If you want `GET()` to take some element literal, wrap it
around `I()` (as you would do in a formula in R). For example, the text
`"123,456"` is replaced with `"123%2C456"`. If you don’t want that
behavior, you would need to do the following `I("123,456")`.

``` r
publications <- GET(
  url   = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi",
  query = list(
    db = "pubmed",
    id = paste(ids, collapse = ","),
    retmax = 300,
    rettype = "abstract"
    )
)

# Turning the output into character vector
publications <- httr::content(publications)
publications_txt <- as.character(publications)
```

With this in hand, we can now analyze the data. This is also a good time
for committing and pushing your work!

## Question 4: Distribution of universities, schools, and departments

Using the function `stringr::str_extract_all()` applied on
`publications_txt`, capture all the terms of the form:

1.  University of …
2.  … Institute of …

Write a regular expression that captures all such instances

Repeat the exercise and this time focus on schools and departments in
the form of

1.  School of …
2.  Department of …

And tabulate the results

## Question 5: Form a database

We want to build a dataset which includes the title and the abstract of
the paper. The title of all records is enclosed by the HTML tag
`ArticleTitle`, and the abstract by `Abstract`.

Before applying the functions to extract text directly, it will help to
process the XML a bit. We will use the `xml2::xml_children()` function
to keep one element per id. This way, if a paper is missing the
abstract, or something else, we will be able to properly match PUBMED
IDS with their corresponding records.

``` r
pub_char_list <- xml2::xml_children(publications)
pub_char_list <- sapply(pub_char_list, as.character)
```

Now, extract the abstract and article title for each one of the elements
of `pub_char_list`. You can either use `sapply()` as we just did, or
simply take advantage of vectorization of `stringr::str_extract`

``` r
abstracts <- str_extract(pub_char_list, "<Abstract>(\\n|.)+</Abstract>")
abstracts <- str_remove_all(abstracts, "</?[[:alnum:]]+>")
abstracts <- str_remove_all(abstracts, "\\s+")
```

- How many of these don’t have an abstract?

``` r
table(is.na(abstracts))
```

    ## 
    ## FALSE  TRUE 
    ##   285    15

15 articles do not have abstracts.

Now, the title

``` r
titles <- str_extract(pub_char_list, "<ArticleTitle>(\\n|.)+</ArticleTitle>")
titles <- str_remove_all(titles, "</?[:alnum:]]+>")
titles <- str_remove_all(titles, "\\s+")
```

- How many of these don’t have a title ?

``` r
table(is.na(titles))
```

    ## 
    ## FALSE 
    ##   300

All articles have titles.

Finally, put everything together into a single `data.frame` and use
`knitr::kable` to print the results

``` r
database <- data.frame(
  PubMedID = ids,
  Title = titles,
  Abstract = abstracts
)
knitr::kable(head(database))
```

| PubMedID | Title                                                                                                                                                                                                                     | Abstract                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|:---------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 39936294 | <ArticleTitle>ClinicalPracticeRecommendationsontheEffectofCOVID-19VaccinationStrategiesonOutcomesinSolidOrganTransplantRecipients.</ArticleTitle>                                                                         | \<AbstractTextLabel=“INTRODUCTION”NlmCategory=“BACKGROUND”\>Solidorgantransplant(SOT)recipientswereexcludedfromclinicaltrialsevaluatingtheefficacyofCOVID-19vaccines.Thereisuncertaintyaboutthenumberofdosesrequiredtopreventlife-threateninginfection,aswellasuncertaintyintheoptimalvaccinetypeandtheirdurability.OurobjectivesweretoproviderecommendationsonthenumberofCOVID-19vaccinationdoses,typeofvaccine,doseofvaccineadministered,andtimingofvaccinationinSOTrecipients.\<AbstractTextLabel=“METHODS”NlmCategory=“METHODS”\>WecommissionedasystematicreviewonCOVID-19vaccinationinSOT,focusingonpatient-importantoutcomes.Werecruitedaninternational,multidisciplinarypanelof18stakeholders,includingpatientpartnerstosummarizeourfindingsusingtheGRADE(gradingofrecommendation,assessment,development,andevaluation)framework,ratecertaintyintheevidence,anddeveloprecommendations.\<AbstractTextLabel=“RESULTS”NlmCategory=“RESULTS”\>OurpanelrecommendstheroutineprovisionofadditionalCOVID-19dosesaftertheprimaryseriestoSOTrecipientswithvariant-appropriatevaccines(strongrecommendation,lowcertaintyevidence).WesuggestusinganyavailableWHO-approvedvaccineratherthanselectivelychoosingaspecifictypeandreceivingasingledoseratherthanadoubledoseofanyCOVID-19vaccinebooster(weakrecommendation,lowcertaintyevidence).Lastly,wesuggestvaccinationbeforetransplantationwhenpossible(weakrecommendation,lowcertaintyevidence).\<AbstractTextLabel=“CONCLUSION”NlmCategory=“CONCLUSIONS”\>TheevidenceusedtoguidetheserecommendationsislimitedbythepaucityofrobustrandomizedtrialsevaluatingCOVID-19vaccinationstrategiesandclinicaloutcomesintheSOTpopulation.Theprovisionofhigher-qualityevidenceoftheoveralleffectsofCOVID-19vaccinationinSOTtoinformclinicalpracticewillrequirelarge,randomizedtrials.©2025JohnWiley&SonsA/S.PublishedbyJohnWiley&SonsLtd.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| 39933692 | <ArticleTitle>AProgramtoReduceEmergencyDepartmentTransfersandBuildLong-TermCareHomeCapacity:AMixed-MethodsStudy.</ArticleTitle>                                                                                           | \<AbstractTextLabel=“OBJECTIVES”NlmCategory=“OBJECTIVE”\>Transferstoacutecarehospitalsexposelong-termcareresidentstopotentialharm.WeimplementedLong-TermCarePlus(LTC+)attheoutsetoftheCOVID-19pandemictoreduceemergencydepartment(ED)transfersandimproveaccesstourgentmedicalservicesbyprovidingvirtualspecialistconsultation,systemnavigation,anddiagnosticandlaboratorytestingto54long-termcarehomes(LTCHs).\<AbstractTextLabel=“DESIGN”NlmCategory=“METHODS”\>Thismixed-methodsstudyaimedtodetermineifLTC+ledtoadecreaseinavoidableacutecaretransfersandtoexploreparticipants’perceptionsandcontextualfactorsinfluencinguptake.\<AbstractTextLabel=“SETTINGANDPARTICIPANTS”NlmCategory=“METHODS”\>LTC+wasimplementedacross54LTCHsand3hospitalhubsinToronto,Canada.\<AbstractTextLabel=“METHODS”NlmCategory=“METHODS”\>StatisticalprocesscontrolchartswerecreatedtodetectchangesinEDtransferrates,stratifyingdataintohigh-andlow-uptakeLTCHstoevaluatetheeffectofLTC+onEDtransferratesacross54LTCHs.Semistructuredinterviewswereconductedwithhealthcareproviders,administrators,residents,andcaregiversacross6LTCHsand3hospitalhubsandanalyzedthematically.\<AbstractTextLabel=“RESULTS”NlmCategory=“RESULTS”\>Therewere9658EDtransfersduringthestudyperiod(April2020toMarch2022),ofwhich3860(40.0%)didnotrequireadmission.LTC+delivered534virtualconsultations,with5LTCHsaccountingfor59%ofprogramuse.Comparedwithbaseline(January2019toFebruary2020),transferratesdecreasedby40%,withnodifferenceseenbetweenLTCHswithhighvslowuptake.Factorsinfluencinguptakeincludeprogramawareness,motivation,alignmentofLTCHresourcesandprogramservices,andcommitmenttoEDavoidance.\<AbstractTextLabel=“CONCLUSIONSANDIMPLICATIONS”NlmCategory=“CONCLUSIONS”\>TheLTC+programdidnotreduceEDtransfersbeyondseculartrendsattributabletothebroadereffectsoftheCOVID-19pandemic.ParticipantsthatusedLTC+identifiedimportantbenefitsthatextendedbeyondEDavoidanceincludingbuildingself-efficacyandcapacityinLTCHstoprovideclient-centeredcarewithcross-sectoralcollaboration.RefinementstotheLTC+programdesignanddeliveryandstructuralchangesareneededtoincreaseimpact.Copyright©2025.PublishedbyElsevierInc.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| 39930162 | <ArticleTitle>EffectofearlyandlaterpronepositioningonoutcomesininvasivelyventilatedCOVID-19patientswithacuterespiratorydistresssyndrome:analysisoftheprospectiveCOVID-19criticalcareconsortiumcohortstudy.</ArticleTitle> | \<AbstractTextLabel=“BACKGROUND”NlmCategory=“BACKGROUND”\>PronepositioningofpatientswithCOVID-19undergoinginvasivemechanicalventilation(IMV)iswidelyused,butevidenceofefficacyremainssparse.TheCOVID-19CriticalCareConsortiumhasgeneratedoneofthelargestglobaldatasetsonthemanagementandoutcomesofcriticallyillCOVID-19patients.Thisprospectivecohortstudyinvestigatedtheassociationbetweenpronepositioningandmortalityandinparticularfocussedontimingoftreatment.\<AbstractTextLabel=“METHODS”NlmCategory=“METHODS”\>Weinvestigatedtheincidence,demographicprofile,managementandoutcomesofpronedpatientsundergoingIMVforCOVID-19inthestudy.Wecomparedoutcomesbetweenpatientspronepositionedwithin48hofIMVtothose(i)neverproned,and(ii)pronedonlyafter48h.\<AbstractTextLabel=“RESULTS”NlmCategory=“RESULTS”\>3131patientshaddataonpronepositioning,1482(47%)wereneverproned,1034(33%)werepronedwithin48hand615(20%)werepronedonlyafter48hofcommencementofIMV.28-day(hazardratio0.82,95%confidenceinterval\[CI\]0.68,0.98,p=0.03)and90-day(hazardratio0.81,95%CI0.68,0.96,p=0.02)mortalityriskswerelowerinthosepatientspronedwithin48hofIMVcomparedtothoseneverproned.However,therewasnoevidenceforastatisticallysignificantassociationbetweenpronepositioningafter48hwith28-day(hazardratio0.93,95%CI0.75,1.14,p=0.47)or90-daymortality(hazardratio0.95,95%CI0.78,1.16,p=0.59).\<AbstractTextLabel=“CONCLUSIONS”NlmCategory=“CONCLUSIONS”\>PronepositioningisassociatedwithimprovedoutcomesinpatientswithCOVID-19,buttimingmatters.Wefoundnoassociationbetweenlaterproningandpatientoutcome.©2025.TheAuthor(s).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 39928659 | <ArticleTitle>ImpactoftheCOVID-19pandemiconoutcomesofacuteischemicstrokepatientstreatedwithendovasculartherapy:AmulticenterCanadianstudy.</ArticleTitle>                                                                  | \<AbstractTextLabel=“OBJECTIVE”NlmCategory=“OBJECTIVE”\>Thenovelcoronavirusdisease2019(COVID-19)pandemicledtotheimplementationofwide-ranginginstitutionalinfectioncontrolprotocols.Thepurposeofthisstudyistodeterminetheeffectofthepandemiconoutcomesoflargevesselocclusion(LVO)acuteischemicstroke(AIS)patientstreatedwithendovasculartherapy(EVT).\<AbstractTextLabel=“MATERIALSANDMETHODS”NlmCategory=“METHODS”\>DatawereobtainedfromprospectivelycollectedqualityimprovementstrokedatabasesatsixCanadiancomprehensivestrokecentresfromMarch11,2020toMarch11,2021.Thispatientcohortwascomparedtopre-pandemicpatientsconsecutivelytreatedwithEVTfromMarch11,2019toMarch10,2020.Theprimaryoutcomeisa90-daymodifiedRankinScore(mRS).Thesecondaryoutcomesareangiographictimemetrics.\<AbstractTextLabel=“RESULTS”NlmCategory=“RESULTS”\>Atotalof1329EVTpatients(pre-pandemicn=666)wereincluded.TheinitialNIHSSwasstatisticallysignificantlylowerinthepandemiccohort.Otherbaselinepatientcharacteristicswerecomparablebetweenthetwoperiods.Median(interquartilerange,IQR)timefromlastseennormal(LSN)toemergencydepartment(ED)(172(68-316)vs210(97-382)min;p=0.0001),LSNtopuncture(235(160-378)vs280(184-475);p\<0.0001),computedtomography(CT)toangiographictable(68(44-108)vs84(57-125)min;p=0.002),EDtoangiographictable(65(37-96)vs80(50-112)min;p=0.001),CTtorecanalization(117(84-156)vs130(89-173)min;p=0.038)andLSNtorecanalization(279(198-453)vs327(219-561)min;p=0.002)werelongerinthepandemicperiodascomparedtothepre-pandemic.Therewerenosignificantdifferencesinmediantimefromangiographictabletoarterialpuncture(13(8-19)vs12(9-16)min;p=0.70)orarterialpuncturetofirstpass(21(14-31)vs20(14-30)min;p=0.50).Patientsweremorelikelytohavefavourableoutcomes(mRSat90daysscoreof≤2)post-EVTpre-pandemicthanpandemic(53%vs44%;p=0.02).Furthermore,analysisofthetimeintervalfrom”LSNtoarterialpuncture”inrelationtofunctionaloutcomesshowedthatthepercentageofunfavorableoutcomesincreasedamongpatientswhounderwentEVTwithin240minutes.Specifically,therateofunfavorableoutcomesrosefrom32.9%to42.9%(p=0.37forintervalsunder150minutes)andfrom41.6%to52.3%(p=0.15forintervalsbetween151and240minutes)whencomparingpre-pandemictopandemicperiods.However,thedetrimentaleffectassociatedwiththepandemicwasdiminishedinpatientswhoreceivedEVTbeyond240mins(p=1.0).\<AbstractTextLabel=“CONCLUSION”NlmCategory=“CONCLUSIONS”\>InthismulticenterstudyinvolvingsixCanadianstrokecenters,patientsexhibitedahigherprobabilityofunfavorablelong-termfunctionaloutcomesfollowingEVTduringthepandemicperiodcomparedtothoseinthepre-pandemiccohort,particularlyduringthefirstyearofthepandemic.Copyright:©2025Zhuetal.ThisisanopenaccessarticledistributedunderthetermsoftheCreativeCommonsAttributionLicense,whichpermitsunrestricteduse,distribution,andreproductioninanymedium,providedtheoriginalauthorandsourcearecredited. |
| 39927080 | <ArticleTitle>DescribingCaregiverandClinicianExperienceswithPediatricTelerehabilitationAcrossClinicalDisciplines.</ArticleTitle>                                                                                          | \<AbstractTextLabel=“SCOPE”NlmCategory=“UNASSIGNED”\>ThisstudydescribesthehighandlowpointsofcaregiverandclinicianexperienceswithpediatrictelerehabilitationwithconsiderationforthesustainableadoptionofpediatrictelerehabilitationbeyondtheCOVID-19pandemiccontext.\<AbstractTextLabel=“METHODS”NlmCategory=“UNASSIGNED”\>Aspartofalargerstudy,thisprojectanalyzeddatafromqualitativeinterviewstodescribecaregivers’(n=27)andclinicians’(n=27)experienceswithpediatrictelerehabilitation.\<AbstractTextLabel=“FINDINGS”NlmCategory=“UNASSIGNED”\>Caregiverandclinicianexperienceswithpediatrictelerehabilitationaredescribedaccordingtofourtouchpointsidentified:(1)childengagementintelerehabilitation;(2)perceivedvalueoftelerehabilitationservicesandcaregiverengagement;(3)preparingthepeopleandenvironmentfortelerehabilitationservices;(4)fitofusingatelerehabilitationmodel;and(5)providingfamilywithchoice.\<AbstractTextLabel=“DISCUSSION”NlmCategory=“UNASSIGNED”\>Findingshighlighttheimportanceofbeinginformedaboutthetelerehabilitationservicemodel,feelingpreparedfortelerehabilitationappointmentsandbeingresponsivetofamilies’choice.Recommendationstoaddresstheseareasarediscussed.Copyright©2025MeaghanReitzel,LoriLetts,CynthiaLennon,JenniferLasenby-Lessard,MonikaNovak-Pavlic,BrianoDiRezze,MichellePhoenix.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 39923009 | <ArticleTitle>EarlyCOVID-19andprotectionfromOmicroninahighlyvaccinatedpopulationinOntario,Canada:amatchedprospectivecohortstudy.</ArticleTitle>                                                                           | \<AbstractTextLabel=“OBJECTIVES”NlmCategory=“OBJECTIVE”\>Predictionsregardingtheon-goingburdenofSARS-CoV-2,andvaccinerecommendations,requireanunderstandingofinfection-associatedimmuneprotection.WeassessedwhetherearlyCOVID-19providedprotectionagainstOmicroninfection.\<AbstractTextLabel=“METHODS”NlmCategory=“METHODS”\>WeenrolledacohortofadultsinOntario,Canada,withCOVID-19priortoOctober2020(earlyinfection,EI),andamatchedcohortwithCOVID-19testingandanegativePCR(non-EI).ParticipantscompletedbaselinesurveysthensurveyseverytwoweeksuntilJanuary2023.MultivariableCoxregressionwasusedtoassessfactorsassociatedwithCOVID-19infectionduringthefirst14monthsofOmicron.\<AbstractTextLabel=“RESULTS”NlmCategory=“RESULTS”\>Overall,624EI(70%)and175(77%)non-EIparticipantsmetcriteriaforanalysis;590(95%)EIand164(94%)non-EIhadreceivedatleast2COVID-19vaccinedosespriortoOmicron.Of624EI,175(28%)hadoneSARS-CoV-2re-infectionand8(1.3%)hadtwo,comparedto84(48%)non-EIparticipantswithone,5(2.9%)withtwoand1(0.6%)with3infections(P\<0.0001).InmultivariableanalysisofriskfactorsforOmicroninfection,theoverallhazardratio(HR,95%CI)associatedwithEIwas0.56(0.43-0.74);HRsforBA.1/2,BA.4/5andmixedBA.5/BQ.1/XBBperiodswere0.66(0.45-0.97),0.44(0.28-0.68)and0.71(0.32-1.56).EIandBA.1/2infectioncombinedreducedlaterOmicroninfection(HR0.07(0.03-0.21)comparedtonopriorinfection.Olderage,non-Whiteethnicity,nochildreninhousehold,andlowerneighbourhoodincomewereassociatedwithreducedriskofinfection.\<AbstractTextLabel=“CONCLUSIONS”NlmCategory=“CONCLUSIONS”\>Inourhighlyvaccinatedpopulation,earlySARS-CoV-2infectionwasassociatedwitha44%reductioninsymptomaticCOVID-19duringthefirst14monthsofOmicron,providingsignificantprotectionagainstre-infectionformorethan2years.©2025.TheAuthor(s).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

Done! Knit the document, commit, and push.

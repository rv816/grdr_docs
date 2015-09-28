
# GRDR Database Documentation
## Overview 
Registries were  added to the "vocabulary" table as though it were a standardized terminology. each registry was given a vocabulary_id (1000-1004), which would be used in other tables to tie questions back to their source registry. 
 <br>
 The **vocabulary** table is critical to understanding the overall structure of this database. In a normal OMOP database, the vocabulary table houses the standard vocabularies used in the OMOP vocabulary. For example, *vocabulary_id* = 1 corresponds to SNOMED-CT, *vocabulary_id*=8 corresponds to RxNorm, etc. 
 <br>
 To keep the data model integrity, we created a "fake" vocabulary out of each registry's questions, and each question was added to the **concept** table as an artificial "concept". While SNOMED CT is *vocabulary_id*=1, BBS Registry's questions were stored at *vocabulary_id*=1000 (we chose out-of-band *vocabulary_id* 's to avoid overlap with the existing standard **vocabulary** table.) This allowed unmapped data to coexist alongside mapped data in the same database. 
 <br>
 Practically, this means that **every** data element from every registry is stored in the database in its original form, with the question text stored in the **grdr_concepts** table under the field *concept_name*, which is assigned a *vocabulary_id * corresponding to the source registry (in the range 1000-1004). In the **observation** table (the primary data table), each question a patient was asked is stored under *observation_concept_id* using the *grdr_concept_id*  from the **grdr_concepts** table. In turn, the **grdr_concepts** table is mirrored in OMOP's  **concept** table to enable the relationships speci
 <br>
 The answer to each question given by each patient is stored in its original form, as submitted by the registries, under *value_as string* in the **observation** table, on a row corresponding to the respective question and patient id. 
 <br>
 For the (large) subset of question-answer pairs that **were mapped**, at least two additional steps were taken. First, a row was created in the **observation_mappings** table for each question (*observation_concept_id*) and answer (*value_as_string*). Second, the mapping(s) of the question was recorded under the set of columns (same row as *observation_concept_id* and *value_as_string*). Most of these mappings were completed in SNOMED CT, and the SNOMED mapping corresponding to a given question was recorded in the groups of columns  including  *omop_concept_id_1* (the assigned primary key for the SNOMED concept in our OMOP database) , *concept_code_1*, (the SNOMED CT ID) *vocabulary_id_1* (SNOMED CT), *preferred_label_1* (+/- *omop_concept_id_2*, *concept_code_2*, etc). 
 <br>
 In some cases, where the patient chose the answer from a discrete, easy to map set of choices (like "Yes"/"No"), the answers to the questions were mapped under the column *value_as_concept_id* in the same row. Unlike questions, which could have multiple mappings that form an expression, answers were mapped to only one SNOMED concept. (The value_as_concept_id can then be joined against the **concept** table on `concept.concept_id = observation_mappings.value_as_concept_id`)
 <br>
 Since mappings are recorded in the **observation_mappings** table and the original data is recorded in the **observation** table, we must execute a left join (**observation** x **observation_mappings**) in order to see the registry data in its mapped form:
```plpgsql
SELECT concept_code_1, vocabulary_id_1, preferred_label_1, value_as_string, dict_maps_extracted, value_as_concept_id
-- [,concept_code_2, vocabulary_id_2, preferred_label_2, concept_code_3, vocabulary_id_3, preferred_label_3]  
FROM observation 
LEFT JOIN observation_mappings on
observation.observation_concept_id = observation_mappings.observation_concept_id 
AND observation.value_as_string = observation_mappings.value_as_string
WHERE TRUE
-- AND  [relevant criteria here ]
``` 

 As an example , if we want to look mapped versions of all questions related to a biopsy (using the SNOMED concept for "Biopsy",  SCTID = 86273004), were can alter this query as follows:
```plpgsql
SELECT concept_code_1, vocabulary_id_1, preferred_label_1, value_as_string, dict_maps_extracted 
-- [,concept_code_2, vocabulary_id_2, preferred_label_2, concept_code_3, vocabulary_id_3, preferred_label_3]  
FROM observation 
LEFT JOIN observation_mappings on
observation.observation_concept_id = observation_mappings.observation_concept_id 
AND observation.value_as_string = observation_mappings.value_as_string
WHERE TRUE
AND  '86273004' in (concept_code_1, concept_code_2, concept_code_3, concept_code_4);

```

## GRDR-Specific Tables

### grdr_concepts
_The core table for GRDR mappings_

All of the data that was sent to us from the registries was normalized into the OMOP data structure as follows. 

For each registry, before we looked at their data, we extracted all of the questions (columns) and added them to a master table with the questions from all registries. This table is called *grdr_concepts*. There is probably too much information in this table for your use (it is designed to fit into the OMOP data model), so in the table below I have identified the columns that should be disregarded.

The **grdr_concepts** table is mirrored in OMOP's  **concept** table,  where the source registry questions sit side by side with standard concepts. This enables unmapped data to be stored in the OMOP data model alongside mapped data.

*public.grdr_concepts*

| Column | Type | Modifiers | Description | 
| ---- | --- | --- | -- | 
| concept_id | integer | not null |  Each question within each registry was given a unique ID whose first four digits were the vocabulary ID of the registry it came from. For example, BBS’s first question was given the ID 10000001  |
| concept_name | character varying(1000) | | This is the text of the question, verbatim, as submitted by the registries
| concept_level | numeric(38,0) |  | **Disregard**. Set to  zero by default to avoid violating non-null constraints in the ""concept" table. | 
| concept_class | character varying(60) |  |   Identifies concept as a raw data element that has not been standardized in contrast with the rest of the  "concept" table.
| vocabulary_id | integer |  | Registries were  added to the "vocabulary" table as though it were a standardized terminology. each registry was given a vocabulary_id (1000-1004), which would be used in other tables to tie questions back to their source registry. |
| concept_code | character varying(40) |  | Also known as "registry_question_name".  this was the label of the column in the flat file with patient data submitted by the registries. |
| valid_start_date | timestamp with time zone | | **Disregard**. Set to  zero by default to avoid violating non-null constraints in the "concept" table. 
| valid_end_date | timestamp with time zone | | **Disregard**. Set to  zero by default to avoid violating non-null constraints in the ""concept" table. 
| invalid_reason | character varying(1) | | **Disregard**. Set to  zero by default to avoid violating non-null constraints in the ""concept" table. 






### observation_mappings
| Column | Type | Description |
| ---- | ---- | ----- |  
| observation_concept_id | integer | Foreign key for source registry question, as it appears in **grdr_concepts** (and mirrored in the **concept** table).  |
| concept_name | character varying(1000) | Question  text, as it appears in the **grdr_concepts** table and mirrored in the **concept** table. | 
| value_as_string | character varying | Each patient's answer to question as submitted by registry (a.k.a. the question's "value"). Since this table is a metadata table, and is meant to store mappings, not patient data, there is one row for each UNIQUE question-answer pair. In contrast, the **observation** table has one row for each answer given by each patient on each question (meaning it is the table with all of the patient data.)  |
| vocabulary_name | character varying(256) | *vocabulary_name* describes the name of the registry.  |
| value_as_concept_id | integer | In some cases, where the patient chose the answer from a discrete, easy to map set of choices (like "Yes"/"No"), the answers to the questions were mapped under the column *value_as_concept_id*. | 
| informationtype | text | Type of answer required by the question. For example, yes/no questions were labeled "bool", whereas free text questions were labeled "freetext", etc. See below for more in depth description of the types.  | 
| patient_count | integer | Total number of patients that answered the question. | 
| id | integer | not null default nextval('observation_mappings_id_seq'::regclass) |  Primary Key for *observation_mappings* table | 
| dict_maps_extracted | jsonb | Some registries recorded numbers to represent certain answers given by the patient (e.g. 1=yes, 0=no). In many cases, the question text was the only place where the meaning of these numeric answers was explained. Using algorithms based on regular expressions, we extracted as many of these "answer keys" as possible. Since these were extracted into the jsonb datatype, this table can only be viewed with PostgreSQL version 9.4 or greater.  | 
| intermediate_string | text | For a given question-answer pair, when "answer keys" were used (e.g. 1=yes, 0=no), this is the string that is represented by the numeric answer recorded by the registry (which, in this table, is stored in *value_as_string*) | 
| snomed_pref_label | text | The preferred label in SNOMED corresponding to the SNOMED code that was mapped to the *intermediate_string*. This field was only used when answers were recorded as numeric values representing string answers (e.g. 1=yes, 0=no.)| 
| snomed_code | text | SNOMED code that was mapped to the *intermediate_string*. This field was only used when answers were recorded as numeric values representing string answers (e.g. 1=yes, 0=no. | 
| question_keywords | text | **Disregard.** Intermediate metadata for automated mapping algorithms. | 
| final_mapping | jsonb | **Disregard.** Intermediate metadata for automated mapping algorithms. | 
| grdr_concept_id | text | **Disregard**. This is the same as *observation_concept_id*, is a (sloppy) remnant of a join. | 
| which | text | **Disregard.** Intermediate metadata for automated mapping algorithms. |
| omop_concept_id_1 | integer |  Mapping of the source registry's question to a standard concept. The assigned primary key for the standard vocabulary concept in our OMOP vocabulary. You can always join anything that ends in "_concept_id" on the **concept** table.| 
| concept_code_1 | text | Mapping of the source registry's question to a standard concept. The concept code represents the identifier of the concept in the source data it originates from, such as SNOMED-CT concept IDs, RxNorm RXCUIs etc. Note that concept codes are not unique across vocabularies.| 
| vocabulary_id_1 | text |  Mapping of the source registry's question to a standard concept. The vocabulary id is a foreign key to the **vocabulary** table indicating from which source the standard concept has been adapted. |
| preferred_label_1 | text | Mapping of the source registry's question to a standard concept. The preferred label is an unambiguous, meaningful and descriptive name for the standard concept.  |
| omop_concept_id_2 | integer | Mapping of the source registry's question to a standard concept. The assigned primary key for the standard vocabulary concept in our OMOP vocabulary. You can always join anything that ends in "_concept_id" on the **concept** table.|
| concept_code_2 | text | Mapping of the source registry's question to a standard concept. The concept code represents the identifier of the concept in the source data it originates from, such as SNOMED-CT concept IDs, RxNorm RXCUIs etc. Note that concept codes are not unique across vocabularies.|
| vocabulary_id_2 | text | Mapping of the source registry's question to a standard concept. The vocabulary id is a foreign key to the **vocabulary** table indicating from which source the standard concept has been adapted. |
| preferred_label_2 | text | Mapping of the source registry's question to a standard concept. The preferred label is an unambiguous, meaningful and descriptive name for the standard concept. |
| domain_id_2 | text ||
| domain_id_1 | text ||
| omop_concept_id_3 | integer | Mapping of the source registry's question to a standard concept. The assigned primary key for the standard vocabulary concept in our OMOP vocabulary. You can always join anything that ends in "_concept_id" on the **concept** table. |
| concept_code_3 | text | Mapping of the source registry's question to a standard concept. The concept code represents the identifier of the concept in the source data it originates from, such as SNOMED-CT concept IDs, RxNorm RXCUIs etc. Note that concept codes are not unique across vocabularies. |
| vocabulary_id_3 | text | Mapping of the source registry's question to a standard concept. The vocabulary id is a foreign key to the **vocabulary** table indicating from which source the standard concept has been adapted. |
| preferred_label_3 | text | Mapping of the source registry's question to a standard concept. The preferred label is an unambiguous, meaningful and descriptive name for the standard concept. |
| domain_id_3 | text ||
| omop_concept_id_4 | integer | Mapping of the source registry's question to a standard concept. The assigned primary key for the standard vocabulary concept in our OMOP vocabulary. You can always join anything that ends in "_concept_id" on the **concept** table. |
| concept_code_4 | text | Mapping of the source registry's question to a standard concept. The concept code represents the identifier of the concept in the source data it originates from, such as SNOMED-CT concept IDs, RxNorm RXCUIs etc. Note that concept codes are not unique across vocabularies. |
| vocabulary_id_4 | text | Mapping of the source registry's question to a standard concept. The vocabulary id is a foreign key to the **vocabulary** table indicating from which source the standard concept has been adapted. |
| preferred_label_4 | text | Mapping of the source registry's question to a standard concept. The preferred label is an unambiguous, meaningful and descriptive name for the standard concept.|
| domain_id_4 | text ||

Indexes:
     "observation_mappings_pkey" PRIMARY KEY, btree (id)
 Foreign-key constraints:
     "observation_mappings_observation_concept_id_fkey" FOREIGN KEY (observation_concept_id) REFERENCES concept(concept_id)


> #### More about "informationtype" and the semantic model for mappings

> **QUESTION TYPE**

> | Abbreviation | Full Name | Short Description | Example Question | Semantic Template |
| ----- | ---- | ---- | ---- | ---- | 
| bool | Boolean | used to label yes-no (or either unknown or inapplicable) responses, alive-deceased (or unknown) responses, and responses that have either been checked or unchecked | Do you have kidney failure? (Yes/No) | {{question}} is boolean:{{answer}} |
| freetext | Free Text | used to label any free response (e.g. a response containing phrases created by the survey taker) | Describe your symptoms freely. (must be free text AND not meet criteria for other categories) | {{question}} is described by patient as free text:{{answer}} |
| garbage | Garbage | used to label responses that do not fit any of the other categories and will not be useful data | {{question}} has unmappable answer garbage:{{answer}} |
| howmany | How Many | used to label numerical responses with units (e.g. inches or an age (in present tense)) | How tall are you (in cm)? (has units) | {{question}} has numeric value:{{answer}} with {{units}} |
| mcterm | Defined Term | used to label responses chosen from a set of defined terms that do not fit any of the other categories | What kind of heart failure did you have? (diastolic/systolic) | {{question}} has attribute:{{answer}} |
| when | When | used to label responses that answer a question related to time (e.g. a date or an age (in past tense)) | When were you first diagnosed with kidney failure? (1945, 1956, 1492) | {{question}} occurred at age/on date:{{answer}} |

> **MODIFIERS**

Certain mappings were compunt

> | Abbreviation | Full Name | Description |
| ---- | ---- | ---- |
| consent | Consent | patient grants permission to doctors/researchers to obtain medical testing results |
| famhx | Family History | medical history of relatives |
| units | Units | standard forms of measuring |

> **More about keywords** 
> Multiple keywords were extracted using an algorithm. Then, the best keyword was selected from the keywords provided by the keyword extraction algorithm. The best keyword was chosen after looking at the question and answer and determining which keyword was the best fit for both.


## The OMOP Common Data Model (GRDR Data)

### observation

The OBSERVATION table captures any clinical facts about a patient obtained in the context of examination, questioning or a procedure. The observation domain supports capture of data not represented by other domains, including unstructured measurements, medical history and family history.

| name | description | required | table | type |
| --- | --- | --- | --- | --- |
| observation_concept_id | A foreign key to the observation concept identifier in the vocabulary. **For the GRDR, this referenced the question text, as stored in the grdr_concepts table and mirrored in the concept table.** | True | observation | integer |
| observation_date | The date of the observation. For many of the registries, the questions asked were of the "Have you ever..." variety, rendering the observation date "unmappable." Accordingly, for most entries, we used an out of band value. <br><br> **Note: Some questions specifically were of the "when" variety--these answers were recorded in _value_as_string_**| True | observation | date |
| observation_id | A system-generated unique identifier for each observation. | True | observation | integer |
| observation_source_value | The observation code as it appears in the source data. **For the GRDR, we stored the source registry column names for each data element in this field.**| False | observation | string |
| value_as_string | **Critical field. All values recorded in registry surveys were stored in this field.**T he observation result stored as a string. This is applicable to observations where the result is expressed as verbatim text, such as in radiology or pathology reports. | False | observation | string |
| observation_time | **Disregard.** The time of the observation. | False | observation | time |
| observation_type_concept_id | A foreign key to the predefined concept identifier in the vocabulary reflecting the type of the observation. **For the GRDR, this was "Patient-reported" for almost all values.**  | True | observation | integer |
| person_id | A foreign key identifier to the person about whom the observation was recorded. The demographic details of that person are stored in the **person** table. **This is a system-generated identifier. The GRDR GUID is stored in the corresponding row in the person table.** | True | observation | integer |
| range_high | **Disregard. **The upper limit of the normal range of the observation. It is not applicable if the observation results are non- numeric or categorical, and must be in the same units of measure as the observation value. | False | observation | number |
| range_low | **Disregard.** The lower limit of the normal range of the observation. It is not applicable if the observation results are non- numeric or categorical, and must be in the same units of measure as the observation value. | False | observation | number |
| relevant_condition_concept_id | **Disregard.** A foreign key to the predefined concept identifier in the vocabulary reflecting the condition that was associated with the observation. Note that this is not a direct reference to a specific condition record in the condition table, but rather a condition concept in the vocabulary. | False | observation | integer |
| unit_concept_id | A foreign key to a standard concept identifier of measurement units in the vocabulary. | False | observation | integer |
| unit_source_value | The source code for the unit as it appears in the source data. This code is mapped to a standard unit concept in the vocabulary and the original code is, stored here for reference. | False | observation | string |
| value_as_concept_id | A foreign key to an observation result stored as a concept identifier. This is applicable to observations where the result can be expressed as a standard concept from the vocabulary (e.g., positive/negative, present/absent, low/high, etc.). | False | observation | integer |
| value_as_number | The observation result stored as a number. This is applicable to observations where the result is expressed as a numeric value. | False | observation | number |
| visit_occurrence_id | **Disregard.** A foreign key to the visit in the visit table during which the observation was recorded. | False | observation | integer |


## condition_occurrence

A diagnosis or condition that has been recorded about a person at a certain time. For this table, a lot of work was commited to standardizing and denormalizing information about the **rare disease diagnosis.** This table is very useful for many of the aggregate analytics that NCATS requires.

| name | description | required | table | type |
| --- | --- | --- | --- | --- |
| associated_provider_id | A foreign key to the provider in the provider table who was responsible for determining (diagnosing) the condition. | False | condition_occurrence | integer |
| condition_concept_id | A foreign key that refers to a standard condition concept identifier in the vocabulary. | True | condition_occurrence | integer |
| condition_end_date | The date when the instance of the condition is considered to have ended. This is not typically recorded. | False | condition_occurrence | date |
| condition_occurrence_id | A system-generated unique identifier for each condition occurrence event. | True | condition_occurrence | integer |
| condition_source_value | The source code for the condition as it appears in the source data. This code is mapped to a standard condition concept in the vocabulary and the original code is, stored here for reference. Condition source codes are typically ICD-9-CM diagnosis codes from medical claims or discharge status/disposition codes from EHRs. | False | condition_occurrence | string |
| condition_start_date | The date when the instance of the condition is recorded. | True | condition_occurrence | date |
| condition_type_concept_id | A foreign key to the predefined concept identifier in the vocabulary reflecting the source data from which the condition was recorded, the level of standardization, and the type of occurrence. Conditions are defined as primary or secondary diagnoses, problem lists and person statuses. | True | condition_occurrence | integer |
| person_id | A foreign key identifier to the person who is experiencing the condition. The demographic details of that person are stored in the person table. | True | condition_occurrence | integer |
| stop_reason | The reason, if available, that the condition was no longer recorded, as indicated in the source data. Valid values include discharged, resolved, etc. | False | condition_occurrence | string |
| visit_occurrence_id | A foreign key to the visit in the visit table during which the condition was determined (diagnosed). | False | condition_occurrence | integer |

### drug_exposure

Association between a Person and a Drug at a specific time. For the GRDR, most drugs were reported as the answers to "have you ever taken..." questions. Accordingly, the notion of time is somewhat irrelevant. Generally, the start date for the drug was date of the survey. For this table, a lot of work was commited to standardizing and denormalizing information about the **rare disease diagnosis.** This table is very useful for many of the aggregate analytics that NCATS requires.

 | name | description | required | table | type |
| --- | --- | --- | --- | --- |
| days_supply | The number of days of supply of the medication as recorded in the original prescription or dispensing record. | False | drug_exposure | number |
| drug_concept_id | A foreign key that refers to a standard concept identifier in the vocabulary for the drug concept. | True | drug_exposure | integer |
| drug_exposure_end_date | The end date for the current instance of drug utilization. It is not available from all sources. | False | drug_exposure | date |
| drug_exposure_id | A system-generated unique identifier for each drug utilization event. | True | drug_exposure | integer |
| drug_exposure_start_date | The start date for the current instance of drug utilization. Valid entries include a start date of a prescription, the date a prescription was filled, or the date on which a drug administration procedure was recorded. | True | drug_exposure | date |
| drug_source_value | The source code for the drug as it appears in the source data. This code is mapped to a standard drug concept in the vocabulary and the original code is, stored here for reference. | False | drug_exposure | string |
| drug_type_concept_id | A foreign key to the predefined concept identifier in the vocabulary reflecting the type of drug exposure recorded. It indicates how the drug exposure was represented in the source data: as medication history, filled prescriptions, etc. | True | drug_exposure | integer |
| person_id | A foreign key identifier to the person who is subjected to the drug. The demographic details of that person are stored in the person table. | True | drug_exposure | integer |
| prescribing_provider_id | A foreign key to the provider in the provider table who initiated (prescribed) the drug exposure. | False | drug_exposure | integer |
| quantity | The quantity of drug as recorded in the original prescription or dispensing record. | False | drug_exposure | number |
| refills | The number of refills after the initial prescription. The initial prescription is not counted, values start with 0. | False | drug_exposure | number |
| relevant_condition_concept_id | A foreign key to the predefined concept identifier in the vocabulary reflecting the condition that was the cause for initiation of the drug exposure. Note that this is not a direct reference to a specific condition record in the condition table, but rather a condition concept in the vocabulary. | False | drug_exposure | integer |
| sig | The directions ("signetur") on the drug prescription as recorded in the original prescription (and printed on the container) or dispensing record. | False | drug_exposure | string |
| stop_reason | The reason the medication was stopped, where available. Reasons include regimen completed, changed, removed, etc. | False | drug_exposure | string |
| visit_occurrence_id | A foreign key to the visit in the visit table during which the drug exposure initiated. | False | drug_exposure | integer |

### care_site

For the GRDR, this was used to track which registry a participant belonged to.

| name |  description | required | table | type |
| --- | --- | --- | --- | --- |
| care_site_id | A system-generated unique identifier for each care site. A care site is the place where the provider delivered the healthcare to the person. | True | care_site | integer |
| care_site_source_value | The identifier for the care site as it appears in the source data, stored here for reference. | False | care_site | string |
| location_id | A foreign key to the geographic location in the location table, where the detailed address information is stored. | False | care_site | integer |
| organization_id | A foreign key to the organization in the organization table, where the detailed information is stored. | False | care_site | integer |
| place_of_service_concept_id | A foreign key to the predefined concept identifier in the vocabulary reflecting the place of service. | False | care_site | integer |
| place_of_service_source_value | The source code for the place of service as it appears in the source data, stored here for reference. | False | care_site | string |




### person

Demographic information about a Person. The **person** table is the core table for the database--all clinical facts are tied to a *person_id*, which is the primary key for patients in the database. 



 | name | description | required | table | type |
| --- | --- | --- | --- | --- |
| care_site_id | **For the GRDR, this field identifies the registry which recorded the data about the patient.** A foreign key to the primary care site in the care site table, where the details of the care site are stored. | False | person | integer |
| day_of_birth | **Disregard.**The day of the month of birth of the person. For data sources that provide the precise date of birth, the day is extracted and stored in this field. | False | person | number |
| ethnicity_concept_id | A foreign key that refers to the standard concept identifier in the vocabulary for the ethnicity of the person. | False | person | integer |
| ethnicity_source_value | The source code for the ethnicity of the person as it appears in the source data. The person ethnicity is mapped to a standard ethnicity concept in the vocabulary and the original code is, stored here for reference. | False | person | string |
| gender_concept_id | A foreign key that refers to a standard concept identifier in the vocabulary for the gender of the person. | True | person | integer |
| gender_source_value | The source code for the gender of the person as it appears in the source data. The person gender is mapped to a standard gender concept in the vocabulary and the corresponding concept identifier is, stored here for reference. | False | person | string |
| location_id | **Disregard.**A foreign key to the place of residency for the person in the location table, where the detailed address information is stored. | False | person | integer |
| month_of_birth | **Disregard.**The month of birth of the person. For data sources that provide the precise date of birth, the month is extracted and stored in this field. | False | person | number |
| person_id | A system-generated unique identifier for each person. | True | person | integer |
| person_source_value | **Critical Field.** For the GRDR, this is where the **GUID** was stored.  <br> <br> In OMOP, this field holds an encrypted key derived from the person identifier in the source data. This is necessary when a drug safety issue requires a link back to the person data at the source dataset. No value with any medical or demographic significance must be stored. | False | person | string |
| provider_id | **Disregard.**A foreign key to the primary care provider the person is seeing in the provider table. | False | person | integer |
| race_concept_id | A foreign key that refers to a standard concept identifier in the vocabulary for the race of the person. | False | person | integer |
| race_source_value | The source code for the race of the person as it appears in the source data. The person race is mapped to a standard race concept in the vocabulary and the original code is, stored here for reference. | False | person | string |
| year_of_birth | The year of birth of the person. For data sources with date of birth, the year is extracted. For data sources where the year of birth is not available, the approximate year of birth is derived based on any age group categorization available. | True | person | number |onl

## The OMOP Standard Vocabulary

### General Notes
#### Version 4 to Version 5
The GRDR had the unfortunate privilege of straddling a major upgrade to the OMOP Data Model and Vocabulary. Largely, this should not present any difficulty in extracting the data from OMOP into a different database, but it results in a somewhat difficult to follow logic model for the database as a standalone entity. 

Broadly, the denormalized tables **condition_occurrence**, **drug_exposure**, **person**, **care_site**, **observation** utilize OMOP V4.  The Standard Vocabulary Tables (**concept**, **vocabulary**, **domain**, **relationship**, **concept_relationship**, **concept_ancestor**, etc) utilize OMOP V5, since V5 provided a richer vocabulary structure (adding the **domain** table, most critically); additionally, V5 contained   more up to date versions of SNOMED CT, RxNorm, LOINC, and others. For almost all users, the difference between OMOP V4 and OMOP V5 will be indistinguishable.


#### Vocabulary ERD

![image](https://cloud.githubusercontent.com/assets/3758548/10145800/50ac9cd0-65f3-11e5-8bfa-496800708c44.png)

______

### By table:

### concept 

OMOP's Standardized Vocabulary contains records, or Concepts, that uniquely identify each fundamental unit of meaning used to express clinical information in all domain tables of the CDM. Concepts are derived from vocabularies, which represent clinical information across a domain (e.g. conditions, drugs, procedures) through the use of codes and associated descriptions. Some Concepts are designated Standard Concepts, meaning these Concepts can be used as normative expressions of a clinical entity within the OMOP Common Data Model and within standardized analytics. Each Standard Concept belongs to one domain, which defines the location where the Concept would be expected to occur within data tables of the CDM. For the GRDR, source registry questions were added to the **concept** table as Concepts. In the OMOP CDM, this enables unmapped data (i.e. data stored in its original question and answer format) to sit side-by-side with mapped data (that is represented by Standard Concepts). 

Generally, Concepts can represent broad categories (like “Cardiovascular disease”), detailed clinical elements (”Myocardial infarction of the anterolateral wall”) or modifying characteristics and attributes that define Concepts at various levels of detail (severity of a disease, associated morphology, etc.).

Records in the Standardized Vocabularies tables are derived from national or international vocabularies such as SNOMED-CT, RxNorm, and LOINC, or custom Concepts defined to cover various aspects of observational data analysis. For a detailed description of these vocabularies, their use in the OMOP CDM and their relationships to each other please refer to the [[documentation:vocabulary|Specifications]].

| Field | Required | Type | Description | 
| ---- | ---- | ---- | ---- |
|concept_id|Yes|integer|A unique identifier for each Concept across all domains.|
|concept_name|Yes|varchar(255)|An unambiguous, meaningful and descriptive name for the Concept.|
|domain_id|Yes|varchar(20)|A foreign key to the [[documentation:cdm:domain|DOMAIN]] table the Concept belongs to.|
|vocabulary_id|Yes|varchar(20)|A foreign key to the [[documentation:cdm:vocabulary|VOCABULARY]] table indicating from which source the Concept has been adapted.|
|concept_class_id|Yes|varchar(20)|The attribute or concept class of the Concept. Examples are “Clinical Drug”, “Ingredient”, “Clinical Finding” etc.|
|standard_concept|No|varchar(1)|This flag determines where a Concept is a Standard Concept, i.e. is used in the data, a Classification Concept, or a non-standard Source Concept. The allowables values are 'S' (Standard Concept) and 'C' (Classification Concept), otherwise the content is NULL.|
|concept_code|Yes|varchar(50)|The concept code represents the identifier of the Concept in the source vocabulary, such as SNOMED-CT concept IDs, RxNorm RXCUIs etc. Note that concept codes are not unique across vocabularies.|
|valid_start_date|Yes|date|The date when the Concept was first recorded. The default value is 1-Jan-1970, meaning, the Concept has no (known) date of inception.|
|valid_end_date|Yes|date|The date when the Concept became invalid because it was deleted or superseded (updated) by a new concept. The default value is 31-Dec-2099, meaning, the Concept is valid until it becomes deprecated.|
|invalid_reason|No|varchar(1)|Reason the Concept was invalidated. Possible values are D (deleted), U (replaced with an update) or NULL when valid_end_date has the default value.|

####  Conventions
Concepts in the Common Data Model are derived from a number of public or proprietary terminologies such as SNOMED-CT and RxNorm, or custom generated to standardize aspects of observational data. Both types of Concepts are integrated based on the following rules:
  * All Concepts are maintained centrally by the CDM and Vocabularies Working Group. Additional concepts can be added, as needed, upon request.
  * For all Concepts, whether they are custom generated or adopted from published terminologies, a unique numeric identifier concept_id is assigned and used as the key to link all observational data to the corresponding Concept reference data. 
  * The concept_id of a Concept is persistent, i.e. stays the same for the same Concept between releases of the Standardized Vocabularies.
  * A descriptive name for each Concept is stored as the Concept Name as part of the **concept** table.
  * Each Concept is assigned to a Domain. For Standard Concepts, these is always a single Domain. Source Concepts can be composite or coordinated entities, and therefore can belong to more than one Domain. The domain_id field of the record contains the abbreviation of the Domain, or Domain combination. Please refer to the Standardized Vocabularies [[documentation:vocabulary:domains|Specification]] for details of the Domain Assignment.
  * Concept Class designation are attributes of Concepts. Each Vocabulary has its own set of permissible Concept Classes, although the same Concept Class can be used by more than one Vocabulary. Depending on the Vocabulary, the Concept Class may categorize Concepts vertically (parallel) or horizontally (hierarchically). 
  * Concept Class attributes should not be confused with Classification Concepts. These are separate Concepts that have a hierarchical relationship to Standard Concepts or each other, while Concept Classes are unique Vocabulary-specific attributes for each Concept.
  * For Concepts inherited from published terminologies, the source code is retained in the concept_code field and can be used to reference the source vocabulary.
*_source_concept_id fields and are not used in CONCEPT_ANCESTOR table. 
  * The lifespan of a Concept is recorded through its valid_start_date, valid_end_date and the invalid_reason fields. This allows Concepts to correctly reflect at which point in time were defined. Usually, Concepts get deprecatd if their meaning was deemed ambigous, a duplication of another Conncept, or needed revision for scientific reason. For example, drug ingredients get updated when different salt or isomer variants enter the market. Usually, drugs taken off the market do not cause a deprecation by the terminology vendor. Since observational data are valid with respect to the time they are recorded, it is key for the Standardized Vocabularies to provide even obsolete codes and maintain their relationships to other current Concepts .
  * Concepts without a known instantiated date are assigned valid_start_date of ‘1-Jan-1970’.
  * Concepts that are not invalid are assigned valid_end_date of ‘31-Dec-2099’.
  * Deprecated Concepts (with a valid_end_date before the release date of the Standardized Vocabularies) will have a value of 'D' (deprecated without successor) or 'U' (updated). 
  * Values for concept_ids generated as part of Standardized Vocabularies will be reserved from 0 to 2,000,000,000. Above this range, concept_ids are available for local use and are guaranteed not to clash with future releases of the Standardized Vocabularies.



### vocabulary

Contains a list of sources for the various Concepts. Many Concepts are derived from national or industry initiatives, such as the ICD diagnostic codes. Others are created by OMOP.

| name |  description | required | table | type |
| --- | --- | --- | --- | --- |
| vocabulary_id | A unique identifier for each vocabulary. | True | vocabulary | integer |
| vocabulary_name | The name describing the vocabulary, for example "SNOMED-CT", "ICD-9", "Visit", etc. | True | vocabulary | string |


### domain

The DOMAIN table includes a list of the domains of data elements that are contained within the OMOP common data model. A domain defines the set of allowable concepts for each standardized field. This reference table is populated with a single record for each domain and includes a descriptive name for the Domain.

 | name | description | required | table | type |
| --- | --- | --- | --- | --- |
| domain_concept_id | A foreign key that refers to a standard concept identifier in the Standardized Vocabularies for the Domain the Domain record belongs to. | True | domain | integer |
| domain_id | A unique key for each domain. | True | domain | string |
| domain_name | The name describing the Domain, e.g. "Condition", "Procedure", "Measurement" etc. | True | domain | string |



### concept_relationship

The **concept_relationship** table contains records that define direct relationships between any two Concepts and the nature or type of the relationship. Each type of a relationship is defined in the **relationship** table.

|Field|Required|Type|Description|
|-----|-----|-----|-------|
|concept_id_1|Yes|integer|A foreign key to a Concept in the [[documentation:cdm:concept|CONCEPT]] table associated with the relationship. Relationships are directional, and this field represents the source concept designation.|
|concept_id_2|Yes|integer|A foreign key to a Concept in the [[documentation:cdm:concept|CONCEPT]] table associated with the relationship. Relationships are directional, and this field represents the destination concept designation.|
|relationship_id|Yes|varchar(20)|A unique identifier to the type or nature of the Relationship as defined in the [[documentation:cdm:relationship|RELATIONSHIP]] table.|
|valid_start_date|Yes|date|The date when the instance of the Concept Relationship is first recorded.|
|valid_end_date|Yes|date|The date when the Concept Relationship became invalid because it was deleted or superseded (updated) by a new relationship. Default value is 31-Dec-2099.|
|invalid_reason|No|varchar(1)|Reason the relationship was invalidated. Possible values are 'D' (deleted), 'U' (replaced with an update) or NULL when valid_end_date has the default value.|

#### Conventions 
  * Relationships can generally be classified as hierarchical (parent-child) or non-hierarchical (lateral). 
  * All Relationships are directional, and each Concept Relationship is represented twice symmetrically within the **concept_relationship**table. For example, the two SNOMED concepts of ‘Acute myocardial infarction of the anterior wall’ and ‘Acute myocardial infarction’ have two Concept Relationships: 1- ‘Acute myocardial infarction of the anterior wall’ ‘Is a’ ‘Acute myocardial infarction’, and 2- ‘Acute myocardial infarction’ ‘Subsumes’ ‘Acute myocardial infarction of the anterior wall’.
  * There is one record for each Concept Relationship connecting the same Concepts with the same relationship_id.
  * Since all Concept Relationships exist with their mirror image (concept_id_1 and concept_id_2 swapped, and the relationship_id replaced by the reverse_relationship_id from the **relationship** table), it is not necessary to query for the existence of a relationship both in the concept_id_1 and concept_id_2 fields.
  * Concept Relationships define direct relationships between Concepts. Indirect relationships through 3rd Concepts are not captured in this table. However, the **concept_ancestor**table does this for hierachical relationships over several "generations" of direct relationships.
  * In previous versions of the CDM, the relationship_id used to be a numerical identifier. See the **relationship** table.



### relationship

The **relationship** table provides a reference list of all types of relationships that can be used to associate any two concepts in the **concept_relationship** table. 

|Field|Required|Type|Description|
|-----|-----|-----|-------|
|relationship_id|Yes|varchar(20)| The type of relationship captured by the relationship record.|
|relationship_name|Yes|varchar(255)| The text that describes the relationship type.|
|is_hierarchical|Yes|varchar(1)|Defines whether a relationship defines concepts into classes or hierarchies. Values are 1 for hierarchical relationship or 0 if not.|
|defines_ancestry|Yes|varchar(1)|Defines whether a hierarchical relationship contributes to the concept_ancestor table. These are subsets of the hierarchical relationships. Valid values are 1 or 0.|
|reverse_relationship_id|Yes|varchar(20)|The identifier for the relationship used to define the reverse relationship between two concepts.|
|relationship_concept_id|Yes|integer|A foreign key that refers to an identifier in the CONCEPT table for the unique relationship concept.|

####Conventions

  * There is one record for each Relationship.
  * Relationships are classified as hierarchical (parent-child) or non-hierarchical (lateral)
  * They are used to determine which concept relationship records should be included in the computation of the CONCEPT_ANCESTOR table.
  * The relationship_id field contains an alphanumerical identifier, that can also be used as the abbreviation of the Relationship.
  * The relationship_name field contains the unabbreviated names of the Relationship.
  * Relationships all exist symmetrically, i.e. in both direction. The relationship_id of the opposite Relationship is provided in the reverse_relationship_id field.
  * Each Relationship also has an equivalent entry in the Concept table, which is recorded in the relationship_concept_id field. This is for purposes of creating a closed Information Model, where all entities in the OMOP CDM are covered by unique Concepts.
  * Hierarchical Relationships are used to build a hierarchical tree out of the Concepts, which is recorded in the CONCEPT_ANCESTOR table. For example, "has_ingredient" is a Relationship between Concepst of the Concept Class 'Clinical Drug' and those of 'Ingredient', and all Ingredients can be classified as the "parental" hierarchical Concepts for the drug products they are part of. All 'Is a' Relationships are hierarchical.
  * Relationships, also hierarchical, can be between Concepts within the same Vocabulary or those adopted from different Vocabulary sources.
  * In past versions of the RELATIONSHIP table, the relationship_id used to be a numerical value. A conversion table between these old and new IDs is given at http://www.ohdsi.org/web/wiki/doku.php?id=documentation:cdm:relationship.


### concept_ancestor 

The CONCEPT_ANCESTOR table is designed to simplify observational analysis by providing the complete hierarchical relationships between Concepts. Only direct parent-child relationships between Concepts are stored in the CONCEPT_RELATIONSHIP table. To determine higher level ancestry connections, all individual direct relationships would have to be navigated at analysis time. The  CONCEPT_ANCESTOR table includes records for all parent-child relationships, as well as grandparent-grandchild relationships and those of any other level of lineage. Using the CONCEPT_ANCESTOR table allows for querying for all descendants of a hierarchical concept. For example, drug ingredients and drug products are all descendants of a drug class ancestor.

This table is entirely derived from the CONCEPT, CONCEPT_RELATIONSHIP and RELATIONSHIP tables.  

| Field| Required | Type | Description |
| ---- | ----- | ----- | ---- |
|ancestor_concept_id|Yes|integer|A foreign key to the concept in the concept table for the higher-level concept that forms the ancestor in the relationship.|
|descendant_concept_id|Yes|integer|A foreign key to the concept in the concept table for the lower-level concept that forms the descendant in the relationship.|
|min_levels_of_separation|Yes|integer|The minimum separation in number of levels of hierarchy between ancestor and descendant concepts. This is an attribute that is used to simplify hierarchic analysis.|
|max_levels_of_separation|Yes|integer|The maximum separation in number of levels of hierarchy between ancestor and descendant concepts. This is an attribute that is used to simplify hierarchic analysis.|

#### Conventions 

  * Each concept is also recorded as an ancestor of itself.
  * Only valid and Standard Concepts participate in the CONCEPT_ANCESTOR table. It is not possible to find ancestors or descendants of deprecated or Source Concepts.
  * Usually, only Concepts of the same Domain are connected through records of the CONCEPT_ANCESTOR table, but there might be exceptions.






RxNorm ing of|
|141|Consists of|
|142|Contained in|
|143|Reformulated in|
|144|Is a|
|145|NDFRT dose form of|
|146|Induced by|
|147|Diagnosed through|
|148|Physiol effect by|
|149|CI physiol effect by|
|150|NDFRT ing of|
|151|CI chem class of|
|152|MoA of|
|153|CI MoA of|
|154|PK of|
|155|May be treated by|
|156|CI by|
|157|May be prevented by|
|158|Metabolite of|
|159|Metabolism of|
|160|Inhibits effect|
|161|Chem structure of|
|162|RxNorm - NDFRT eq|
|163|Recipient cat of|
|164|Proc site of|
|165|Priority of|
|166|Pathology of|
|167|Part of|
|168|Severity of|
|169|Revision status of|
|170|Access of|
|171|Occurrence of|
|172|Method of|
|173|Laterality of|
|174|Interprets of|
|175|Indir morph of|
|176|Indir device of|
|177|Specimen of|
|178|Interpretation of|
|179|Intent of|
|180|Focus of|
|181|Manifestation of|
|182|Active ing of|
|183|Finding site of|
|184|Episodicity of|
|185|Dir subst of|
|186|Dir morph of|
|187|Dir device of|
|188|Component of|
|189|Causative agent of|
|190|Asso morph of|
|191|Asso finding of|
|192|Measurement of|
|193|Property of|
|194|Scale type of|
|195|Time aspect of|
|196|Specimen proc of|
|197|Specimen identity of|
|198|Specimen morph of|
|199|Specimen topo of|
|200|Specimen subst of|
|201|Due to of|
|202|Relat context of|
|203|Dose form of|
|204|Occurs before|
|205|Asso proc of|
|206|Dir proc site of|
|207|Indir proc site of|
|208|Proc device of|
|209|Proc morph of|
|210|Finding context of|
|211|Proc context of|
|212|Temporal context of|
|213|Asso with finding|
|214|Surgical appr of|
|215|Device used by|
|216|Energy used by|
|217|subst used by|
|218|Acc device used by|
|219|Clinical course of|
|220|Route of admin of|
|221|Finding method of|
|222|Finding inform of|
|226|SNOMED - ICD9P eq|
|227|SNOMED cat - CPT4|
|228|SNOMED - CPT4 eq|
|239|SNOMED - MedDRA eq|
|240|Is FDA-appr ind of|
|241|Is off-label ind of|
|243|Is CI of|
|244|RxNorm - ETC|
|245|RxNorm - ATC|
|246|MedDRA - SMQ|
|247|Ind/CI - SNOMED|
|248|SNOMED - ind/CI|
|275|Has therap class|
|276|Therap class of|
|277|Drug-drug inter for|
|278|Has drug-drug inter|
|279|Has pharma prep|
|280|Pharma prep in|
|281|Inferred class of|
|282|Has inferred class|
|283|SNOMED proc - HCPCS|
|284|HCPCS - SNOMED proc|
|285|RxNorm - NDFRT name|
|286|NDFRT - RxNorm name|
|287|ETC - RxNorm name|
|288|RxNorm - ETC name|
|289|ATC - RxNorm name|
|290|RxNorm - ATC name|
|291|HOI - SNOMED|
|292|SNOMED - HOI|
|293|DOI - RxNorm|
|294|RxNorm - DOI|
|295|HOI - MedDRA|
|296|MedDRA - HOI|
|297|NUCC - CMS Specialty|
|298|CMS Specialty - NUCC|
|299|DRG - MS-DRG eq|
|300|MS-DRG - DRG eq|
|301|DRG - MDC cat|
|302|MDC cat - DRG|
|303|Visit cat - PoS|
|304|PoS - Visit cat|
|305|VAProd - NDFRT|
|306|NDFRT - VAProd|
|307|VAProd - RxNorm eq|
|308|RxNorm - VAProd eq|
|309|RxNorm replaced by|
|310|RxNorm replaces|
|311|SNOMED replaced by|
|312|SNOMED replaces|
|313|ICD9P replaced by|
|314|ICD9P replaces|
|315|Multilex has ing|
|316|Multilex ing of|
|317|RxNorm - Multilex eq|
|318|Multilex - RxNorm eq|
|319|Multilex ing - class|
|320|Class - Multilex ing|
|321|Maps to|
|322|Mapped from|
|325|Map includes child|
|326|Included in map from|
|327|Map excludes child|
|328|Excluded in map from|
|345|UCUM replaced by|
|346|UCUM replaces|
|347|Concept replaced by|
|348|Concept replaces|
|349|Concept same_as to|
|350|Concept same_as from|
|351|Concept alt_to to|
|352|Concept alt_to from|
|353|Concept poss_eq to|
|354|Concept poss_eq from|
|355|Concept was_a to|
|356|Concept was_a from|
|357|SNOMED meas - HCPCS|
|358|HCPCS - SNOMED meas|
|359|Domain subsumes|
|360|Is domain|



### concept_ancestor

The CONCEPT_ANCESTOR table is designed to simplify observational analysis by providing the complete hierarchical relationships between Concepts. Only direct parent-child relationships between Concepts are stored in the CONCEPT_RELATIONSHIP table. To determine higher level ancestry connections, all individual direct relationships would have to be navigated at analysis time. The  CONCEPT_ANCESTOR table includes records for all parent-child relationships, as well as grandparent-grandchild relationships and those of any other level of lineage. Using the CONCEPT_ANCESTOR table allows for querying for all descendants of a hierarchical concept. For example, drug ingredients and drug products are all descendants of a drug class ancestor.

This table is entirely derived from the CONCEPT, CONCEPT_RELATIONSHIP and RELATIONSHIP tables.  

| Field| Required | Type | Description |
| ---- | ----- | ----- | ---- |
|ancestor_concept_id|Yes|integer|A foreign key to the concept in the concept table for the higher-level concept that forms the ancestor in the relationship.|
|descendant_concept_id|Yes|integer|A foreign key to the concept in the concept table for the lower-level concept that forms the descendant in the relationship.|
|min_levels_of_separation|Yes|integer|The minimum separation in number of levels of hierarchy between ancestor and descendant concepts. This is an attribute that is used to simplify hierarchic analysis.|
|max_levels_of_separation|Yes|integer|The maximum separation in number of levels of hierarchy between ancestor and descendant concepts. This is an attribute that is used to simplify hierarchic analysis.|

#### Conventions

  * Each concept is also recorded as an ancestor of itself.
  * Only valid and Standard Concepts participate in the CONCEPT_ANCESTOR table. It is not possible to find ancestors or descendants of deprecated or Source Concepts.
  * Usually, only Concepts of the same Domain are connected through records of the CONCEPT_ANCESTOR table, but there might be exceptions.

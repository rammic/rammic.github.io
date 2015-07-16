---
layout: post
title: "Medication Identifiers and How to Check for Potential Problems"
description: "Medication Identifiers and How to Check for Potential Problems"
category: development
tags: [pillfill,prescription,rxnorm,ndfrt,ndc,spl,nih,rxnav]
---
{% include JB/setup %}

Let's take a look at the various identifiers in this post. We’ll also review how you can use that data to check for related warning & alerts using the [PillFill API services](https://developer.pillfill.com).

#Prescription Structure
Let’s begin by looking at an example of a prescription collected by the PillFill RX aggregator (with some fields omitted):

{% highlight json %}
{
  "rxNumber": "995575",
  "medicationName": "OMEPRAZOLE DR 40 MG CAPSULE",
  "pharmacyStoreId": "CVS #1183",
  "daysSupply": 30,
  "quantityRemaining": 30,
  "quantityPerDose":1,
  "dosesPerDay":1,
  "dispenseDate": "2014-04-30",
  "computedInactiveAfterDate": "2013-04-29",
  "previousDispenseDates": [
       "2013-04-29",
       "2013-03-01"
  ],
  "brandNames": [
       "Prilosec 40 MG Delayed Release Oral Capsule"
   ],
   "quantity": "30",
   "ndc": "00378522293",
   "rxNormId": "200329",
   "ndfrtNui": "N0000156769",
   "ndfrtName": "OMEPRAZOLE 40MG CAP,SA",
   "splId": "44260509-A91C-4906-BCC7-4EB5D3465DED",
   "uuid": "00153170CDEC5571782287505711C59EB63C",
}
{% endhighlight %}

The three most important identifiers included in the prescription data are:

* The FDA's *National Drug Code* (ndc)
* The FDA's *Structured Product Label* (splId)
* NIH's *RxNorm ID* (RxNormId)

We also make liberal use of: 

* The VA's *National Drug File Reference Terminology* ID  (NUI)
* The FDA's *UNique Ingredient Identifier* (UNII)


#Identifier Relationships

The natural question to ask at this point is *"Why so many identifiers?"* Each identification system provides different levels of understanding to what the medication actually is — a question that can be surprisingly hard to answer at times. Let’s consider Ibuprofen as an example- we have to consider it from multiple perspectives:

* **Drug Ingredients**: Which drugs are included / what are the active ingredients? (e.g. Ibuprofen, Acetaminophen & Caffeine)
* **Drug Concept**: What’s the strength & form of the drug? (e.g. 200mg Ibuprofen pill, 500mg Acetaminophen pill)
* **Drug Package**: How is the drug packaged or group together? Who manufactured it? (e.g. Equate 500ct Ibuprofen, Equate 200ct Ibuprofen 2-Pack, Equate 12ct Ibuprofen Convenience Pack)
* **Drug Product**: Which one of the specific drug packages did the patient receive? (Equate 12ct Ibuprofen Convenience Pack)

Each of the above identifiers can help answer those questions:

![](/img/drug-relationships.png)

A very general hierarchy for OTC Ibuprofen (slightly different for RX drugs)

* Ingredient = UNique Ingredient Identifier (UNII)
* Concept = RxNorm ID
* Package = Structured Product Label (SPL)
* Product = National Drug Code (NDC)

Like with most hierarchal relationships, it’s important for accuracy to start with the most specific identifier available when trying to figure out where something belongs. That’s why PillFill focuses on gathering NDCs whenever possible, since all other relationships can be easily and accurately derived from there. If the NDC is not available, PillFill will associate at least RxNorm/NDFRT identifiers for each prescription.

#Medication Warnings & Alerts
So now that we have a working understanding of the medication identifiers, how can we use them to check for potential problems and gain additional insights?

**FDA Recalls & Shortage Alerts**

The FDA issues recall & shortage alerts primarily based on NDCs. PillFill provides a RESTFul service for checking for such alerts. It’s as simple as you might hope- a HTTP GET call to an endpoint with the 11-Digit NDC from the prescription:

`https://developer.pillfill.com/service/v1/alerts/fdaAlerts?ids=00603388821`

If the drug is found to have any alerts, you’ll get them back in the result set:

{% highlight json %}
{ 
	"ndc": ["00603388821"], 
	"type": "recall", 
	"reason": [ "Qualitest Issues Voluntary, Nationwide Recall for One Lot of Hydrocodone Bitartrate and Acetaminophen Tablets, USP 10 mg/500 mg Due to the Potential for Oversized Tablets" ], 
	"resolution": [], 
	"additionalInfoUrl":  "http://www.fda.gov/Safety/Recalls/ucm318827.htm"
}
{% endhighlight %}

#Drug Ingredient Overdose Warnings

The FDA has published a Maximum Recommended Therapeutic Dose (MRTD) guideline which identifies the maximum amount of each drug ingredient is considered safe based on the weight of the individual (using mg/kg). PillFill offers another RESTFul service to handle these calculations for solid oral (pill-based) medications:

`https://developer.pillfill.com/service/v1/interactions/mrtd?ids=[RX-ID_0]&ids=[RX-ID_1]…&weightInKgs=68`

For this service to operate correctly, the prescription must have the SplId available and the quantityPerDose / dosesPerDay fields set for each prescription included. The service will then calculate the ingredient dose levels across all products and provide feedback if the value is over 90% of the MRTD level.

{% highlight json %}
{
     "unii": "KG60484QX9",
     "ingredientName": "Omeprazole",
     "currentLoad": 91,
     "mrtd": 2,
     "relatedRxs": ["RX-ID_0","RX-ID_1"]
}
{% endhighlight %}

#Drug/Drug Interactions

To check for potential drug interactions, we’re going to use a NIH-provided RESTful service. It requires the RxNorm ID of each drug to be considered. Again, it’s a relatively simple RESTful GET request to find potential interactions:

`http://rxnav.nlm.nih.gov/REST/interaction/list.json?rxcuis=207106+152923+656659`

When an interaction is found, a response is generated with some (fairly basic) detail about why the interaction is relevant:

{% highlight json %}
{
	"fullInteractionType":[
	{
		"minConcept":[
		{
			"rxcui":"152923",
			"name":"Simvastatin 40 MG Oral Tablet [Zocor]","tty":"SBD"
		},
		{
			"rxcui":"656659",
			"name":"bosentan 125 MG Oral Tablet","tty":"SCD"
		}],
		"description":"Bosentan may decrease the serum concentration of simvastatin by increasing its metabolism. Monitor for changes in the therapeutic and adverse effects of simvastatin if bosentan is initiated, discontinued or dose changed."}
	},{
		"minConcept":[
		{
			"rxcui":"152923",
			"name":"Simvastatin 40 MG Oral Tablet [Zocor]",
			"tty":"SBD"
		},
		{
			"rxcui":"207106",
			"name":"Fluconazole 50 MG Oral Tablet [Diflucan]",
			"tty":"SBD"
		}],
		"description":"Increased risk of myopathy/rhabdomyolysis"
	}]
}
{% endhighlight %}

#Summary

Each identifier is useful in different ways, especially considering each service you’ll want to use will require levels of specificity. Once you understand how each interrelates though, you’ll realize the power associated with each of the different identifier and their terminology/information models.

That said, there are countless other checks you can also preform given the specific information found in the prescription- Ingredient allergies using UNIIs, Drug & Condition/Disease Contraindication Checks, etc. Each can help a patient potentially avoid a serious problem that they otherwise would have never considered otherwise. For PHRs to ever go mainstream, they must and should help to that end. It’s no longer enough to simply track pills.
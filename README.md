SELECT
    ARTICLE_TITLE_CLEAN,
    PRIORITY_TIER,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    ARTICLE_URL
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2
WHERE PRIORITY_TIER = 'MONITOR'
ORDER BY SIGNAL_SCORE DESC, RECEIVED_TS_PARSED DESC NULLS LAST
LIMIT 50;

[FinalPharmaNews1.2_2026-06-03-1049.csv](https://github.com/user-attachments/files/28543411/FinalPharmaNews1.2_2026-06-03-1049.csv)
ARTICLE_TITLE_CLEAN,PRIORITY_TIER,SIGNAL_SCORE,SIGNAL_REASONS,MATCHED_COMPANIES,MATCHED_COMPANY_CATEGORIES,ARTICLE_URL
"The Oral GLP-1 Tracker: Amid Foundayo growing pains, Jefferies thinks blockbuster 1st year 'doable'",MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej72fjxyxARa80%7EgjB9%7CYf2p
ASCO: Revolution Medicines confident in RAS leadership as rivals square up,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej0-fjxyxARa80%7EgjB9%7CYf2p
Executive Editor: Eric Sagonowsky,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejA2fjxyxARa
Rallybio swerves past Candid pothole to land deal with cancer drug developer Avenzo,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej8%5EfjxyxARa80%7EgjB9%7CYf2p
Staff Writers: Zoey Becker,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejBmfjxyxARa
Lilly pens $1.2B deal for Hanmi’s GLP-2 drug being aimed at short bowel syndrome,MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej02fjxyxARa80%7EgjB9%7CYf2p
Agios signs $165M deal for blood disorder drug from Korea’s Oscotec that flunked phase 2 study,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej8qfjxyxARa80%7EgjB9%7CYf2p
ASCO: Revolution Medicines confident in RAS leadership as rivals square up,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoejzyfjxyxARa80%7EgjB9%7CYf2p
Publisher: Rebecca Willumson,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejB2fjxyxARa
Oncology solutions from Labcorp,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoej-efjxyxARa
Editor-in-Chief: Ayla Ellison,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejByfjxyxARa
"Lilly locks in 5-program R&D pact with China’s Haisco worth up to $3B, but targets unclear",MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej06fjxyxARa80%7EgjB9%7CYf2p
Lilly pens $1.2B deal for Hanmi’s GLP-2 drug,MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoejzqfjxyxARa80%7EgjB9%7CYf2p
Shionogi's COVID antiviral Xocova passes muster with FDA as post-exposure preventative,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoejz-fjxyxARa80%7EgjB9%7CYf2p
Servier inks $2.6B buyout of Edgewise’s muscular dystrophy unit,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoejz2fjxyxARa80%7EgjB9%7CYf2p
Servier inks $2.6B buyout of Edgewise’s muscular dystrophy unit to beef up neurology pipeline,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej0DfjxyxARa80%7EgjB9%7CYf2p
Associate Editor: Fraiser Kansteiner,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejBefjxyxARa
Healthcare is hard. 43North invests $1M in teams built for it.,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoej-mfjxyxARa
Senior Editors: Ben Adams,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejA6fjxyxARa
Deputy Editor: Angus Liu,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejADfjxyxARa
ASCO: Lilly ties Retevmo to ‘dramatic’ outcomes in early-stage lung cancer with rare RET biomarker,MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej8%7CfjxyxARa80%7EgjB9%7CYf2p
Senior Writer: Kevin Dunleavy,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoejB%7CfjxyxARa
An insider’s look at LillyDirect,MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej9efjxyxARa80%7EgjB9%7CYf2p
Lilly locks in 5-program R&D pact,MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoejz%5EfjxyxARa80%7EgjB9%7CYf2p
"Boston, China and the future of biotech",MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej82fjxyxARa80%7EgjB9%7CYf2p
Healthcare is hard. 43North invests $1M in teams built for it.,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesfaece8nLoej96fjxyxARa
BioAge CEO talks NLRP3 and ‘pipeline in a pill’ ambitions,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoej8-fjxyxARa80%7EgjB9%7CYf2p
"The Oral GLP-1 Tracker: Amid Foundayo growing pains, Jefferies thinks blockbuster 1st year 'doable'",MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesfaece8nLoejzDfjxyxARa80%7EgjB9%7CYf2p
"A year ago, biotech was in its bleakest stretch in a decade. Now IPOs are rebounding and China deals are heating up. John Carroll and a panel of industry leaders dig into the latest Endpoints 100 results and uncover what’s next for the sector.",MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-yd/
Sh­iono­gi wins US ap­proval for first pill to pre­vent Covid fol­low­ing ex­po­sure,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-z/
Servier signs $1.55B upfront deal for Edgewise's muscular dystrophy assets,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-o/
"Summit, Akeso drug reduces death by 34% in China lung cancer study. Here’s what it means",MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-x/
The cost of a PRV is twice as high as it was three years ago — and it’s likely to stay that way,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-d/
"FDA's next fee deal, stacked with US incentives, is under White House review",MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-c/
Oculis’ eye drops for di­a­bet­ic mac­u­lar ede­ma flunk Phase 3 test,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-ju/
The cost of a PRV is twice as high as it was three years ago — and it’s like­ly to stay that way,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-a/
Shionogi wins US approval for first pill to prevent Covid following exposure,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-h/
"Sum­mit, Ake­so drug re­duces death by 34% in Chi­na lung can­cer study. Here’s what it means",MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-jh/
Servi­er signs $1.55B up­front deal for Edge­wise's mus­cu­lar dy­s­tro­phy as­sets,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-yu/
Japan's Sh­iono­gi on Mon­day said it won FDA ap­proval,MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-v/
"Yes, Rev­o­lu­tion Med­i­ci­nes' pan­cre­at­ic can­cer da­ta are that good",MONITOR,0,,,,https://e.endpointsnews.com/t/t-l-whijjll-tkukirkro-jj/
ASCO: Sac-TMT’s massive phase 3 program has a jarring gap. Does Merck plan to close it?,MONITOR,0,,Merck,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesjLece8s6oefMqfjwzvfRa80%7EgjB9%7CYf2p
Sales Director: Angelique Alcover,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesjLece8s6oefQ%7CfjwzvfRa
ASCO: Akeso’s ivonescimab bests PD-1 inhibitor in lung cancer chemo combos,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesjLece8s6oefC6fjwzvfRa80%7EgjB9%7CYf2p
Senior Editor: Ben Adams,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesjLece8s6oefP%5EfjwzvfRa
An insider’s look at LillyDirect,MONITOR,0,,Lilly,Top 25,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesjLece8s6oefNyfjwzvfRa80%7EgjB9%7CYf2p
ASCO: Revolution Medicines confident in RAS leadership as rivals square up,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesjLece8s6oefD6fjwzvfRa80%7EgjB9%7CYf2p
ASCO: Revolution Medicines confident in RAS leadership as rivals square up,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesjLece8s6oefM%5EfjwzvfRa80%7EgjB9%7CYf2p
"Legend scientific founder returns to ASCO with new ambition for high-yield, non-gene-editing CAR-T platform",MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/uc%5EcmQ6cesjLece8s6oefMyfjwzvfRa80%7EgjB9%7CYf2p
How HER2 biology is shaping next-generation cancer treatment,MONITOR,0,,,,https://qtx.omeclk.com/portal/wts/ug%5EcmQ6cesjLece8s6oefFyfjwzvfRa

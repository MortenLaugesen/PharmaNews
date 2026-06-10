AI_COMPLETE(
            'llama3.3-70b',
            CONCAT(
                'You are summarizing pharmaceutical and biopharmaceutical industry news for a Business Intelligence and Insights team at a biologics/CDMO company. ',

                'Summarize only the news item that matches the selected headline below in 1-3 short paragraphs. ',
                'The source text may contain multiple newsletter items, article teasers, links, advertisements, author bios, or unrelated stories. ',
                'Ignore unrelated stories, even if they mention the same company, same disease area, same event type, or same industry. ',
                'Do not include information from other headlines, other article teasers, related news sections, or surrounding newsletter content. ',

                'The summary should help senior stakeholders quickly understand why the story matters. ',
                'Focus on specifics related to monetary value, capability expansion or reduction, and how the change is expected to shape the industry. ',

                'Use only details that clearly belong to the selected headline. ',
                'Do not simply restate the headline. ',
                'Do not invent facts, figures, manufacturing relevance, CDMO relevance, competitive implications, strategic implications, or industry impact that are not supported by the source text. ',

                'When available, include specific company names, deal values, investment amounts, facility or site names, geography, modality, capacity, product or asset names, disease area, and timeline. ',

                'Paragraph 1 should summarize what happened. ',
                'Paragraph 2 should explain the business, capability, competitive, manufacturing, regulatory, or market implication when supported by the source text. ',
                'Paragraph 3 may be used only if there is enough source text to explain broader industry impact. ',

                'If no monetary value is stated for the selected headline, briefly say that no specific monetary value was stated. ',
                'If no capability expansion or reduction is stated for the selected headline, briefly say that no specific capability change was stated. ',
                'Only say that limited detail was available when the source text truly contains little more than the headline or teaser. ',

                'Write in a professional, concise style suitable for senior stakeholders. ',
                'No bullet points. No quotation marks. ',
                'Target length: 150-260 words when sufficient source text is available. ',
                'If the source text is very limited, write a shorter but still useful summary instead of forcing length. ',

                'Selected headline: ', COALESCE(ARTICLE_TITLE, ''), '. ',
                'Source: ', COALESCE(EMAIL_SOURCE_TYPE, ''), '. ',
                'Date: ', COALESCE(TO_VARCHAR(PUBLISH_DATE), 'Unknown'), '. ',
                'Matched companies: ', COALESCE(MATCHED_COMPANIES, ''), '. ',
                'Company categories: ', COALESCE(MATCHED_COMPANY_CATEGORIES, ''), '. ',
                'Signal reasons: ', COALESCE(SIGNAL_REASONS, ''), '. ',

                'Source text: ',
                COALESCE(LEFT(ARTICLE_SUMMARY_INPUT, 9000), '')
            ),
            {
                'temperature': 0,
                'max_tokens': 900
            }
        ) AS AI_SUMMARY,

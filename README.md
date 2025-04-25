# GoogleAds_Data_Dashboard
# Business Background
While working, as the account/data manager, on a digital advertising project for a senior living marketplace in the US, a dashboard was needed to share performace internally as well as wih the the client. The data team at the client side provided us with the looker dashboard consisting of all the important post-lead metrics that we needed to track. 
I was responsible guiding the client in creating the Google Ads dashboard and creating the link between each dollar spent, each keyword we bid on to each lead and its quality.

# Challenge
Integrating Google Ads data with client's CRM (leads) data to understand impact of the campaigns at each click/keyword level.

### Creating Metadata for campaigns
```sql
SELECT
  DISTINCT(campaign_id),
  campaign_name,
  campaign_advertising_channel_type,
  campaign_advertising_channel_sub_type,
  campaign_start_date,
  campaign_serving_status,
FROM
  `aroscop-456222.Seniorly_Adwords.ads_Campaign_9925610920`
WHERE
  campaign_start_date >'2025-02-25'
```

### Getting metrics & dimensions for Campaigns
```sql
SELECT
  segments_date,
  Campaign.campaign_id,
  Meta.campaign_name,
  Meta.campaign_advertising_channel_type,
  Campaign.metrics_impressions,
  Campaign.metrics_clicks,
  Campaign.metrics_conversions,
  Campaign.segments_device,
  round(Campaign.metrics_cost_micros/1000000,2) as cost
FROM
  `aroscop-456222.Seniorly_Adwords.ads_CampaignBasicStats_9925610920` AS Campaign
LEFT JOIN
  `Cleaned_Data_Metadata_Seniorly.Campaign_Meta` AS Meta
USING
  (campaign_id)
ORDER BY
segments_date
```
### Clicks Ids & related metrics 
```sql
Select
  click_view_gclid,
  Clicks.campaign_id,
  ad_group_id,
  metrics_clicks,
  click_view_page_number,
  click_view_keyword,
  click_view_keyword_info_match_type,
  click_view_keyword_info_text,
  click_view_location_of_presence_city,
  click_view_location_of_presence_metro,
  click_view_location_of_presence_most_specific,
  customer_descriptive_name,
  segments_click_type,
  segments_device,
  segments_date
FROM
  `Seniorly_Adwords.ads_ClickStats_9925610920` AS Clicks
LEFT OUTER
JOIN
`Cleaned_Data_Metadata_Seniorly.Campaign_Meta`
using(campaign_id)
ORDER BY
  segments_date
```

### Keyword level information
```sql
SELECT
  ad_group_id,
  campaign_id,
  Meta.campaign_name,
  ad_group_criterion_criterion_id,
  ad_group_criterion_keyword_match_type,
  ad_group_criterion_keyword_text,
  ad_group_criterion_negative,
  ad_group_criterion_position_estimates_estimated_add_clicks_at_first_position_cpc,
  ad_group_criterion_position_estimates_estimated_add_cost_at_first_position_cpc,
  ad_group_criterion_position_estimates_first_page_cpc_micros,
  ad_group_criterion_quality_info_post_click_quality_score
FROM
  `aroscop-456222.Seniorly_Adwords.p_ads_Keyword_9925610920` as Keywords
LEFT JOIN
  `Cleaned_Data_Metadata_Seniorly.Campaign_Meta` as Meta
  Using (campaign_id)
Where
 Meta.campaign_start_date >='2025-02-25'
```

### Search queries (accept those that have been acted upon (added to keywords or negatives)

```sql
SELECT
  sqt.segments_date,
  sqt.campaign_id,
  cmp.campaign_name,
  sqt.ad_group_id,
  sqt.ad_group_ad_ad_id,
  sqt.metrics_impressions,
  sqt.metrics_clicks,
  sqt.metrics_conversions,
  sqt.metrics_all_conversions,
  sqt.metrics_cost_micros/1000000 AS cost,
  sqt.segments_device,
  sqt.search_term_view_status,
  sqt.search_term_view_search_term
FROM
  `Seniorly_Adwords.ads_SearchQueryStats_9925610920` AS sqt
LEFT JOIN
  `Cleaned_Data_Metadata_Seniorly.Campaign_Meta`AS cmp
USING
  (campaign_id)
WHERE
  cmp.campaign_start_date > '2025-02-25'
  AND sqt.search_term_view_status = 'NONE'
```

### Appending campaign meta data (as new campaigns launch)
``` sql
INSERT INTO
`Cleaned_Data_Metadata_Seniorly.Campaign_Meta`
SELECT
  campaign_id,
  campaign_name,
  campaign_advertising_channel_type,
  campaign_advertising_channel_sub_type,
  campaign_start_date,
  campaign_serving_status,
FROM
  `Seniorly_Adwords.p_ads_Campaign_9925610920` AS Campaign
WHERE
  NOT EXISTS (
  SELECT
    campaign_id
  FROM
    `aroscop-456222.Cleaned_Data_Metadata_Seniorly.Campaign_Meta`
  WHERE
    campaign_id = Campaign.campaign_id )
  AND Campaign.campaign_start_date >'2025-02-25'
  AND Campaign.campaign_name <> **********************  (duplicate campaign name)
```

### Appending campaign metrics

```sql
Insert INTO
`Cleaned_Data_Metadata_Seniorly.Campaign_Metrics`
SELECT
  segments_date,
  Campaign.campaign_id,
  Meta.campaign_name,
  Meta.campaign_advertising_channel_type,
  Campaign.metrics_impressions,
  Campaign.metrics_clicks,
  Campaign.metrics_conversions,
  Campaign.segments_device,
  round(Campaign.metrics_cost_micros/1000000,2) as cost
FROM
  `aroscop-456222.Seniorly_Adwords.ads_CampaignBasicStats_9925610920` AS Campaign
LEFT JOIN
  `Cleaned_Data_Metadata_Seniorly.Campaign_Meta` AS Meta
USING
  (campaign_id)
Where Campaign.segments_date > (select max(segments_date) from `Cleaned_Data_Metadata_Seniorly.Campaign_Metrics`)
ORDER BY
segments_date
```

### Appending Clicks_Gclid metrics

```sql
INSERT INTO
`Cleaned_Data_Metadata_Seniorly.Clicks_Gclids`
Select
  click_view_gclid,
  Clicks.campaign_id,
  ad_group_id,
  metrics_clicks,
  click_view_page_number,
  click_view_keyword,
  click_view_keyword_info_match_type,
  click_view_keyword_info_text,
  click_view_location_of_presence_city,
  click_view_location_of_presence_metro,
  click_view_location_of_presence_most_specific,
  customer_descriptive_name,
  segments_click_type,
  segments_device,
  segments_date
FROM
  `Seniorly_Adwords.ads_ClickStats_9925610920` AS Clicks
LEFT OUTER
JOIN
`Cleaned_Data_Metadata_Seniorly.Campaign_Meta`
using(campaign_id)
Where Clicks.segments_date > (select max(segments_date) from `Cleaned_Data_Metadata_Seniorly.Clicks_Gclids`)
AND
not exists
(select * from `Cleaned_Data_Metadata_Seniorly.Clicks_Gclids` as Old
WHERE
  Old.click_view_gclid = Clicks.click_view_gclid
)
ORDER BY
  Clicks.segments_date
```

### Append Search Queries data

```sql
INSERT INTO
`Cleaned_Data_Metadata_Seniorly.Search_Queries`
SELECT
  sqt.segments_date,
  sqt.campaign_id,
  cmp.campaign_name,
  sqt.ad_group_id,
  sqt.ad_group_ad_ad_id,
  sqt.metrics_impressions,
  sqt.metrics_clicks,
  sqt.metrics_conversions,
  sqt.metrics_all_conversions,
  sqt.metrics_cost_micros/1000000 AS cost,
  sqt.segments_device,
  sqt.search_term_view_status,
  sqt.search_term_view_search_term
FROM
  `Seniorly_Adwords.ads_SearchQueryStats_9925610920` AS sqt
LEFT JOIN
  `Cleaned_Data_Metadata_Seniorly.Campaign_Meta`AS cmp
USING
  (campaign_id)
WHERE
  cmp.campaign_start_date > '2025-02-25'
  AND sqt.search_term_view_status = 'NONE'
  AND sqt.segments_date > (select max(segments_date) from `Cleaned_Data_Metadata_Seniorly.Search_Queries`)
```

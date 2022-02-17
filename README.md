# Trendyol_Bootcamp_HW1

Q1:
-- --> Promosyon çıkılmış fakat hiç satılmamış ürünleri tespit edebilir miyiz?
with n_s as (select distinct a.v2ProductName as name_unsold, 
from `data-to-insights.ecommerce.web_analytics` wa 
    inner join (select * from `data-to-insights.ecommerce.all_sessions` ) a on a.visitId=wa.visitId
    cross join unnest(wa.hits) h 
    cross join unnest (h.promotion) p
    where a.eCommerceAction_type !='6'
    ),
    s as (select distinct a.v2ProductName as name_sold, 
from `data-to-insights.ecommerce.web_analytics` wa 
    inner join (select * from `data-to-insights.ecommerce.all_sessions` ) a on a.visitId=wa.visitId
    cross join unnest(wa.hits) h 
    cross join unnest (h.promotion) p
    where a.eCommerceAction_type ='6'
    ),
     mix as (select name_unsold,name_sold from n_s left join s on( n_s.name_unsold = s.name_sold)
     where name_sold is null
     )
    
SELECT * from mix


Q2:

--Mart, nisan, mayıs aylarında ziyaretçilerin 
--en çok görüntülediği 
--fakat satın alınmamış ürünlere ihtiyacımız var. 
--Her ayın top 10 ürününü gösterebilir misiniz? 
--(Tarih yıl - ay olarak gösterilmeli.)
with m_u_mart as ( select '2017-03' as Dates, v2ProductName,count(pageviews) as total_views_unsold,  from `data-to-insights.ecommerce.all_sessions`
where date like '____03%'  and eCommerceAction_type != '6' 
group by v2ProductName
),
m_s_mart as( select  v2ProductName,count(pageviews) as total_views_sold,  from `data-to-insights.ecommerce.all_sessions`
where date like '____03%'  and eCommerceAction_type = '6' 
group by v2ProductName
), 
mix_mart as( 
select '2017-03' as Dates, v2ProductName,total_views_unsold,total_views_sold  from m_u_mart left join m_s_mart using(v2ProductName)
where total_views_sold is null
),m_u_nisan as ( select '2017-04' as Dates, v2ProductName,count(pageviews) as total_views_unsold,  from `data-to-insights.ecommerce.all_sessions`
where date like '____04%'  and eCommerceAction_type != '6' 
group by v2ProductName
),
m_s_nisan as( select  v2ProductName,count(pageviews) as total_views_sold,  from `data-to-insights.ecommerce.all_sessions`
where date like '____04%'  and eCommerceAction_type = '6' 
group by v2ProductName
), 
mix_nisan as( 
select '2017-04' as Dates, v2ProductName,total_views_unsold,total_views_sold  from m_u_nisan left join m_s_nisan using(v2ProductName)
where total_views_sold is null
),m_u_mayis as ( select '2017-05' as Dates, v2ProductName,count(pageviews) as total_views_unsold,  from `data-to-insights.ecommerce.all_sessions`
where date like '____05%'  and eCommerceAction_type != '6' 
group by v2ProductName
),
m_s_mayis as( select  v2ProductName,count(pageviews) as total_views_sold,  from `data-to-insights.ecommerce.all_sessions`
where date like '____05%'  and eCommerceAction_type = '6' 
group by v2ProductName
), 
mix_mayis as( 
select '2017-05' as Dates, v2ProductName,total_views_unsold,total_views_sold  from m_u_mayis left join m_s_mayis using(v2ProductName)
where total_views_sold is null
)
select * from (
select '2017-03' as Dates, v2ProductName,total_views_unsold,total_views_sold,row_number() over(partition by Dates order by total_views_unsold desc) as rank from mix_mart
UNION ALL
select '2017-04' as Dates, v2ProductName,total_views_unsold,total_views_sold,row_number() over(partition by Dates order by total_views_unsold desc  ) as rank from mix_nisan
UNION ALL
select '2017-05' as Dates, v2ProductName,total_views_unsold,total_views_sold,row_number() over(partition by Dates order by total_views_unsold desc  ) as rank from mix_mayis
ORDER BY Dates, total_views_unsold desc
)
where rank <11
limit 30;


Q3:
-- --> E ticaret sitemiz için günün bölümlerinde, en fazla ilgi gören kategorileri öğrenmek istiyoruz.
            select * from( 
             select part_of_day,
                      v2ProductCategory,
                        total_point*first_visit_score as total_overall_score,
                          row_number() over (partition by part_of_day order by total_point desc) as rank from
            (   select   
                  CASE WHEN h.hour BETWEEN 5 and 9 then 'Early Morning'
                       WHEN h.hour BETWEEN 9 and 12 then 'Late Morning'
                       WHEN h.hour BETWEEN 12 and 15 then 'Early Afternoon'
                       WHEN h.hour BETWEEN 15 and 18 then 'Late Afternoon'
                       WHEN h.hour BETWEEN 18 and 22 then 'Evening'
                       ELSE 'Night'
                       end as Part_Of_Day,p.v2ProductCategory,CASE WHEN  count(totals.newVisits) is not null then 10
                                                                    ELSE  1
                                                                    end as first_visit_score,count(h.hitNumber)*count(p.v2ProductCategory) total_point  
              from `data-to-insights.ecommerce.web_analytics` wa
              CROSS JOIN UNNEST(wa.hits) h
              CROSS JOIN UNNEST(h.product) p
              WHERE p.v2ProductCategory != '(not set)' 
              group by 1,2              
))
where rank <4
order by  Part_Of_Day,rank,total_overall_score desc

# Problem Set: Social Media Analytics Platform

**CSCI 112 - Contemporary Databases**  
**Topic:** MongoDB Data Modeling - Performance Optimization Patterns  
**Format:** Oral Examination

---

## Scenario

You are designing a MongoDB database for a social media analytics platform that tracks engagement metrics for posts across multiple social networks. The platform serves marketing agencies who need real-time dashboards showing post performance metrics and automated daily reports.

The system handles millions of posts daily, with each post receiving hundreds to thousands of interactions (likes, shares, comments) throughout its lifecycle. Marketing teams frequently query for engagement summaries, trending metrics, and performance comparisons across different time periods and demographics.

---

## Dataset Categories

### Posts Collection
- Post metadata (title, content, platform, timestamp)
- Author information and follower counts
- Engagement metrics and performance indicators
- Content categorization and hashtags

### Interactions Collection  
- Individual user interactions with posts
- Interaction type, timestamp, and user demographics
- Engagement quality scores and sentiment data
- Geographic and device information

### Analytics Collection
- Aggregated performance summaries
- Platform-wise statistics
- Trending content analysis
- Historical comparison data

---

## Sample Data

```json
// Posts
{
  "_id": "post_001",
  "title": "Summer Fashion Trends 2024",
  "platform": "instagram",
  "author": "fashionista_jane",
  "follower_count": 45000,
  "created_at": "2024-03-15T10:30:00Z",
  "hashtags": ["fashion", "summer", "trends"],
  "category": "lifestyle",
  "engagement": {
    "interaction_count": 2847,
    "engagement_rate": 4.2,
    "last_updated": "2024-03-15T18:45:00Z"
  }
}

// Interactions
{
  "_id": "int_001",
  "post_id": "post_001",
  "user_id": "user_12345",
  "type": "like",
  "timestamp": "2024-03-15T11:15:00Z",
  "user_demographics": {
    "age_group": "25-34",
    "location": "US"
  },
  "quality_score": 0.8
}

// Analytics Summary
{
  "_id": "analytics_2024-03-15",
  "date": "2024-03-15",
  "platform": "instagram",
  "total_posts": 15420,
  "avg_engagement_rate": 3.7,
  "top_category": "lifestyle",
  "interaction_breakdown": {
    "likes": 125000,
    "shares": 45000,
    "comments": 32000
  }
}
```

---

## Task

1. **Design the Schema:** Create an efficient MongoDB schema for the social media analytics platform that optimizes for both real-time engagement tracking and analytical reporting.

2. **Create Aggregation Pipeline:** Design an aggregation pipeline that processes the interaction data to generate the daily analytics summaries shown in the sample data above.

---

## Guide Questions

1. How would you structure the post documents to minimize computational overhead when displaying engagement metrics on dashboards that serve thousands of concurrent users?

2. What aggregation pipeline stages would you use to calculate daily platform statistics from millions of interaction records, and how would you optimize pipeline performance?

3. As the system scales to handle billions of interactions per day, how would you modify your approach to updating engagement metrics while maintaining acceptable response times?

4. When posts receive millions of interactions, what strategy would you implement to balance between maintaining accurate engagement rates and system performance?

5. How would you handle scenarios where exact engagement calculations become too expensive, and what level of accuracy would be acceptable for different types of analytics queries?

6. What approach would you take when the cost of running your aggregation pipeline daily becomes prohibitive due to data volume growth?

7. How would you decide when to transition from exact calculations to approximate values for high-volume posts, and what factors would influence this decision?

8. What implementation strategy would you use to reduce the frequency of engagement metric updates for viral posts without significantly impacting the user experience?

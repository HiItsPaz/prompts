# Loox Reddit Monitoring - Complete Make.com + Google Sheets Implementation Guide

## Overview

This guide provides complete technical specifications for implementing the 5-Phase Reddit Monitoring System using **Make.com** and **Google Sheets**. Follow this step-by-step to build the entire automated system.

**Tech Stack:**
- **Automation**: Make.com (4 scenarios)
- **Data Storage**: Google Sheets (5 interconnected sheets)
- **AI Analysis**: OpenAI API integration
- **Reddit Integration**: Reddit API via HTTP modules

---

## Google Sheets Structure

### **Master Spreadsheet: "Loox Reddit Monitor"**

Create a single Google Sheets file with 5 tabs:

#### **Sheet 1: Raw_Posts**
```
Column A: Post_ID (Reddit post ID)
Column B: Title (Post title)
Column C: Content (Post content/body)
Column D: Author (Reddit username)
Column E: Subreddit (r/subreddit)
Column F: Post_URL (Direct link to post)
Column G: Author_Karma (User's comment karma)
Column H: Post_Karma (Upvotes on this post)
Column I: Created_Date (When post was created)
Column J: Detected_Date (When our system found it)
Column K: Keywords_Matched (Which keywords triggered detection)
Column L: Status (New, Processing, Analyzed, Filtered_Out)
Column M: Processing_Notes (Any notes from automation)
```

#### **Sheet 2: Scored_Posts**
```
Column A: Post_ID (Links back to Raw_Posts)
Column B: Opportunity_Score (1-10 final score)
Column C: Intent_Score (1-10)
Column D: Urgency_Score (1-10)
Column E: Authority_Score (1-10)
Column F: Priority (High, Medium, Low)
Column G: Context_Category (solution_seeking, complaint, comparison, general_discussion)
Column H: Business_Relevance (1-10)
Column I: Competitor_Mentioned (judge.me, yotpo, okendo, stamped, none)
Column J: Pain_Points (JSON array of identified pain points)
Column K: AI_Analysis_Raw (Full JSON response from OpenAI)
Column L: Recommendation (proceed, skip, monitor)
Column M: Analysis_Date (When analysis completed)
Column N: Status (New, Ready_for_Response, Skipped)
```

#### **Sheet 3: Response_Queue**
```
Column A: Queue_ID (Unique identifier)
Column B: Post_ID (Links to Raw_Posts and Scored_Posts)
Column C: Response_Text (Generated response content)
Column D: Strategy_Used (direct_recommendation, helpful_positioning, etc.)
Column E: Template_Category (A, B, C, D)
Column F: Authenticity_Score (AI-generated quality score)
Column G: Assigned_Account (Which account will post)
Column H: Scheduled_Time (When to post)
Column I: Status (Pending_Review, Approved, Posted, Failed)
Column J: Human_Review_Flag (TRUE/FALSE)
Column K: Review_Notes (Human reviewer comments)
Column L: Generated_Date (When response was created)
Column M: Approved_Date (When human approved)
Column N: Manual_Edits (Any changes made by human)
```

#### **Sheet 4: Posted_Responses**
```
Column A: Response_ID (Unique identifier)
Column B: Post_ID (Original Reddit post)
Column C: Comment_ID (Reddit comment ID of our response)
Column D: Account_Used (Which account posted)
Column E: Subreddit (Where it was posted)
Column F: Posted_Date (When posted)
Column G: Response_Text (Final text that was posted)
Column H: Initial_Upvotes (Upvotes after 2 hours)
Column I: Final_Upvotes (Upvotes after 24 hours)
Column J: Replies_Received (Number of replies)
Column K: Engagement_Score (Calculated engagement metric)
Column L: Community_Reaction (Positive, Neutral, Negative)
Column M: Conversion_Tracked (If we detected business outcome)
Column N: Notes (Any observations)
```

#### **Sheet 5: Account_Health**
```
Column A: Account_Name (Reddit username)
Column B: Account_Type (Primary, Secondary, Support)
Column C: Health_Score (1-10 current health)
Column D: Karma_Total (Current total karma)
Column E: Karma_Trend_7d (Change over 7 days)
Column F: Last_Post_Date (Most recent post)
Column G: Posts_Today (Number of posts today)
Column H: Posts_This_Week (Number of posts this week)
Column I: Warnings_Count (Mod warnings received)
Column J: Status (Active, Paused, Recovery, Banned)
Column K: Cooldown_Until (When account can post again)
Column L: Subreddit_Karma (JSON of karma per subreddit)
Column M: Last_Updated (When health was last checked)
Column N: Risk_Level (Low, Medium, High)
```

---

## Make.com Scenario 1: Reddit Listener

**Purpose:** Monitor Reddit every 12 hours for relevant posts
**Trigger:** Schedule (12:00 PM and 12:00 AM daily)
**Runtime:** ~5-10 minutes

### **Module Flow:**

#### **1. Schedule Module**
```
Settings:
- Trigger: Schedule
- Interval: Every 12 hours  
- Times: 12:00 and 00:00
- Timezone: Your timezone
- Start Date: Today
```

#### **2. HTTP Module: Reddit Search (r/shopify)**
```
Settings:
- URL: https://www.reddit.com/r/shopify/search.json
- Method: GET
- Query String:
  - q: review app OR shopify reviews OR judge.me OR yotpo OR loox
  - sort: new
  - t: day
  - limit: 25
  - restrict_sr: 1

Headers:
- User-Agent: Loox Monitor Bot 1.0

Parse Response: Yes
```

#### **3. Iterator Module**
```
Settings:
- Array: {{2.data.children}}
```

#### **4. Router Module**
```
Route 1: New Posts Filter
Route 2: Error Handling
```

#### **5A. Filter Module: Check if New Post**
```
Condition: 
{{3.data.name}} Does not equal any value from Google Sheets Raw_Posts Column A

Logic: AND
Continue processing: Only if condition is met
```

#### **5B. Filter Module: Keywords Match**
```
Keywords to Check (OR logic):
- review app
- review platform  
- shopify reviews
- judge.me
- yotpo
- okendo
- stamped
- fera
- social proof
- customer reviews
- review widget

Text to Search: {{3.data.title}} + {{3.data.selftext}}
Case Sensitive: No
```

#### **6. JavaScript Module: Extract Data**
```javascript
// Clean and structure the Reddit post data
const postData = input.data;

// Extract author karma (with error handling)
let authorKarma = 0;
try {
    if (postData.author_flair_css_class) {
        // Try to extract karma from flair or other sources
        authorKarma = parseInt(postData.author_flair_text) || 0;
    }
} catch (e) {
    authorKarma = 0;
}

// Identify matched keywords
const title = (postData.title || '').toLowerCase();
const content = (postData.selftext || '').toLowerCase();
const combinedText = title + ' ' + content;

const keywords = [
    'review app', 'review platform', 'shopify reviews', 
    'judge.me', 'yotpo', 'okendo', 'stamped', 'fera',
    'social proof', 'customer reviews', 'review widget'
];

const matchedKeywords = keywords.filter(keyword => 
    combinedText.includes(keyword)
);

return {
    post_id: postData.name,
    title: postData.title,
    content: postData.selftext,
    author: postData.author,
    subreddit: 'r/shopify',
    post_url: 'https://reddit.com' + postData.permalink,
    author_karma: authorKarma,
    post_karma: postData.ups || 0,
    created_date: new Date(postData.created_utc * 1000).toISOString(),
    detected_date: new Date().toISOString(),
    keywords_matched: matchedKeywords.join(', '),
    status: 'New',
    processing_notes: 'Detected by Reddit Listener'
};
```

#### **7. Google Sheets: Add Row (Raw_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Raw_Posts
- Values:
  Row 1: {{6.post_id}}
  Row 2: {{6.title}}
  Row 3: {{6.content}}
  Row 4: {{6.author}}
  Row 5: {{6.subreddit}}
  Row 6: {{6.post_url}}
  Row 7: {{6.author_karma}}
  Row 8: {{6.post_karma}}
  Row 9: {{6.created_date}}
  Row 10: {{6.detected_date}}
  Row 11: {{6.keywords_matched}}
  Row 12: {{6.status}}
  Row 13: {{6.processing_notes}}
```

#### **8-11. Repeat HTTP Modules for Other Subreddits**
```
Module 8: HTTP for r/ecommerce
Module 9: HTTP for r/entrepreneur  
Module 10: HTTP for r/dropshipping
Module 11: HTTP for r/smallbusiness

(Same configuration as Module 2, just change the subreddit in URL)
```

#### **12. Error Handler Module**
```
Settings:
- Directive: Resume
- Resume: Next module

Actions on Error:
- Log error to Google Sheets error log
- Send notification email
- Continue with next subreddit
```

---

## Make.com Scenario 2: Opportunity Scorer

**Purpose:** Analyze new posts from Raw_Posts and score opportunities
**Trigger:** Google Sheets watch rows (Raw_Posts)
**Runtime:** ~2-3 minutes per post

### **Module Flow:**

#### **1. Google Sheets: Watch Rows (Raw_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Raw_Posts
- Trigger Column: Status
- Trigger Value: New
- Limit: 1 (process one at a time)
```

#### **2. Filter Module: Layer 1 Keyword Filter**
```javascript
// Advanced keyword filtering
const title = ({{1.title}} || '').toLowerCase();
const content = ({{1.content}} || '').toLowerCase();
const text = title + ' ' + content;

// Positive keywords (must have at least 1)
const positive = [
    'review app', 'review platform', 'review software',
    'shopify reviews', 'ecommerce reviews', 'product reviews',
    'social proof', 'customer feedback', 'review management',
    'judge.me', 'yotpo', 'okendo', 'stamped', 'fera',
    'alternative', 'recommendation', 'better than',
    'fake reviews', 'review problems', 'low conversion',
    'review widget', 'review display', 'review collection'
];

// Negative keywords (auto-reject)
const negative = [
    'amazon reviews', 'google reviews', 'yelp reviews',
    'app store reviews', 'facebook reviews',
    'movie review', 'book review', 'restaurant review',
    'performance review', 'code review', 'peer review',
    'free only', 'no budget', 'zero cost', 'completely free'
];

const hasPositive = positive.some(keyword => text.includes(keyword));
const hasNegative = negative.some(keyword => text.includes(keyword));

// Pass to next stage if positive AND not negative
return hasPositive && !hasNegative;
```

#### **3. OpenAI: Layer 2 Context Analysis**
```
Settings:
- Model: gpt-3.5-turbo
- Max Tokens: 200
- Temperature: 0.3

System Message:
"You are an expert at analyzing Reddit posts for business opportunities. Analyze posts for context, intent, and business relevance."

User Message:
"Analyze this Reddit post for business opportunity:

SUBREDDIT: {{1.subreddit}}
TITLE: {{1.title}}
CONTENT: {{1.content}}
AUTHOR KARMA: {{1.author_karma}}

Respond ONLY with this JSON format:
{
  \"context_category\": \"complaint|solution_seeking|comparison|general_discussion\",
  \"business_context\": \"ecommerce|software|marketing|other\",
  \"pain_points\": [\"specific pain point 1\", \"pain point 2\"],
  \"solution_intent\": \"high|medium|low|none\",
  \"competitor_mentioned\": \"judge.me|yotpo|okendo|stamped|none\",
  \"urgency_indicators\": [\"needs asap\", \"researching now\", \"future consideration\"],
  \"authenticity_score\": 1-10,
  \"business_relevance\": 1-10,
  \"recommendation\": \"proceed|skip|monitor\"
}

DO NOT include explanations, only return the JSON."
```

#### **4. JSON Parser Module**
```
Settings:
- JSON String: {{3.choices[0].message.content}}
```

#### **5. Filter Module: Layer 2 Pass Criteria**
```
Conditions (AND logic):
- {{4.business_relevance}} Greater than or equal to 6
- {{4.authenticity_score}} Greater than or equal to 5
- {{4.recommendation}} Equal to "proceed"
```

#### **6. JavaScript Module: Layer 3 Intent Scoring**
```javascript
// Calculate intent, urgency, and authority scores
const aiData = {
    context_category: '{{4.context_category}}',
    solution_intent: '{{4.solution_intent}}',
    pain_points: {{4.pain_points}},
    urgency_indicators: {{4.urgency_indicators}},
    competitor_mentioned: '{{4.competitor_mentioned}}',
    business_relevance: parseInt('{{4.business_relevance}}'),
    authenticity_score: parseInt('{{4.authenticity_score}}')
};

const postData = {
    title: '{{1.title}}',
    content: '{{1.content}}',
    author_karma: parseInt('{{1.author_karma}}') || 0,
    post_karma: parseInt('{{1.post_karma}}') || 0,
    subreddit: '{{1.subreddit}}'
};

// Intent Score (1-10)
let intentScore = 0;
if (aiData.context_category === 'solution_seeking') intentScore += 4;
if (aiData.pain_points.length >= 2) intentScore += 2;
if (aiData.urgency_indicators.includes('needs asap')) intentScore += 3;
if (aiData.urgency_indicators.includes('researching now')) intentScore += 2;
if (aiData.competitor_mentioned !== 'none') intentScore += 1;
if (postData.title.toLowerCase().includes('alternative')) intentScore += 1;
intentScore = Math.min(intentScore, 10);

// Urgency Score (1-10)
let urgencyScore = 2; // baseline
const text = (postData.title + ' ' + postData.content).toLowerCase();
if (text.match(/asap|urgent|quickly|soon/)) urgencyScore += 3;
if (text.match(/this week|today|now/)) urgencyScore += 4;
if (text.match(/researching|comparing|evaluating/)) urgencyScore += 2;
if (text.match(/eventually|someday|future/)) urgencyScore -= 1;
if (text.match(/launching|going live|deadline/)) urgencyScore += 2;
if (postData.subreddit === 'r/entrepreneur') urgencyScore += 1;
urgencyScore = Math.max(1, Math.min(urgencyScore, 10));

// Authority Score (1-10)
let authorityScore = 5; // baseline
if (postData.author_karma > 1000) authorityScore += 2;
if (postData.author_karma > 5000) authorityScore += 1;
if (postData.post_karma > 5) authorityScore += 1;
if (text.match(/my store|my business|my company/)) authorityScore += 2;
if (text.match(/our team|we use|we need/)) authorityScore += 1;
if (postData.subreddit.includes('shopify') || postData.subreddit.includes('ecommerce')) authorityScore += 1;
if (postData.author_karma < 50) authorityScore -= 2;
authorityScore = Math.max(1, Math.min(authorityScore, 10));

// Final Opportunity Score (weighted)
const finalScore = (intentScore * 0.5) + (urgencyScore * 0.3) + (authorityScore * 0.2);

// Priority assignment
let priority = 'Low';
if (finalScore >= 8) priority = 'High';
else if (finalScore >= 6) priority = 'Medium';

return {
    opportunity_score: Math.round(finalScore * 10) / 10,
    intent_score: intentScore,
    urgency_score: urgencyScore,
    authority_score: authorityScore,
    priority: priority,
    analysis_date: new Date().toISOString(),
    ai_analysis_raw: JSON.stringify(aiData)
};
```

#### **7. Google Sheets: Add Row (Scored_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Scored_Posts
- Values:
  Row 1: {{1.post_id}}
  Row 2: {{6.opportunity_score}}
  Row 3: {{6.intent_score}}
  Row 4: {{6.urgency_score}}
  Row 5: {{6.authority_score}}
  Row 6: {{6.priority}}
  Row 7: {{4.context_category}}
  Row 8: {{4.business_relevance}}
  Row 9: {{4.competitor_mentioned}}
  Row 10: {{4.pain_points}}
  Row 11: {{6.ai_analysis_raw}}
  Row 12: {{4.recommendation}}
  Row 13: {{6.analysis_date}}
  Row 14: Ready_for_Response
```

#### **8. Google Sheets: Update Row (Raw_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Raw_Posts
- Row Number: {{1.__row_number__}}
- Column L (Status): Analyzed
- Column M (Processing_Notes): Scored {{6.opportunity_score}}, Priority: {{6.priority}}
```

---

## Make.com Scenario 3: Response Generator

**Purpose:** Generate authentic responses for high-scoring opportunities
**Trigger:** Google Sheets watch rows (Scored_Posts)
**Runtime:** ~3-5 minutes per response

### **Module Flow:**

#### **1. Google Sheets: Watch Rows (Scored_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Scored_Posts
- Trigger Column: Status
- Trigger Value: Ready_for_Response
- Limit: 1
```

#### **2. Filter Module: Priority Check**
```
Conditions (OR logic):
- {{1.priority}} Equal to "High"
- {{1.priority}} Equal to "Medium" AND {{1.opportunity_score}} Greater than 6.5
```

#### **3. Google Sheets: Get Values (Raw_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Raw_Posts
- Range: A:M
- Filter: Column A (Post_ID) = {{1.post_id}}
- Return: Single Row
```

#### **4. JavaScript Module: Strategy Selection**
```javascript
// Determine response strategy based on analysis
const scoredData = {
    intent_score: parseFloat('{{1.intent_score}}'),
    urgency_score: parseFloat('{{1.urgency_score}}'),
    authority_score: parseFloat('{{1.authority_score}}'),
    context_category: '{{1.context_category}}',
    competitor_mentioned: '{{1.competitor_mentioned}}',
    priority: '{{1.priority}}'
};

const postData = {
    title: '{{3.title}}',
    content: '{{3.content}}',
    subreddit: '{{3.subreddit}}'
};

// Strategy selection logic
let strategy = 'educational';
let templateCategory = 'D';

if (scoredData.intent_score >= 8 && scoredData.context_category === 'solution_seeking') {
    strategy = 'direct_recommendation';
    templateCategory = 'A';
} else if (scoredData.intent_score >= 6 && scoredData.urgency_score >= 6) {
    strategy = 'helpful_positioning';
    templateCategory = 'B';
} else if (scoredData.context_category === 'comparison') {
    strategy = 'balanced_comparison';
    templateCategory = 'C';
}

// Extract key variables for personalization
const painPoint = extractPainPoint(postData.content);
const userContext = extractUserContext(postData.content);

function extractPainPoint(content) {
    const painIndicators = [
        'problem', 'issue', 'trouble', 'struggling', 'difficult',
        'not working', 'broken', 'slow', 'expensive', 'complicated'
    ];
    
    for (const indicator of painIndicators) {
        if (content.toLowerCase().includes(indicator)) {
            return indicator;
        }
    }
    return 'challenge';
}

function extractUserContext(content) {
    if (content.toLowerCase().includes('my store')) return 'store_owner';
    if (content.toLowerCase().includes('our business')) return 'business_owner';
    if (content.toLowerCase().includes('client')) return 'consultant';
    return 'entrepreneur';
}

return {
    strategy: strategy,
    template_category: templateCategory,
    pain_point: painPoint,
    user_context: userContext,
    competitor_mentioned: scoredData.competitor_mentioned,
    subreddit_style: getSubredditStyle(postData.subreddit)
};

function getSubredditStyle(subreddit) {
    const styles = {
        'r/shopify': 'direct_business_focused',
        'r/ecommerce': 'strategic_analytical', 
        'r/entrepreneur': 'experience_story_driven',
        'r/dropshipping': 'roi_practical'
    };
    return styles[subreddit] || 'general_helpful';
}
```

#### **5. OpenAI: Response Generation**
```
Settings:
- Model: gpt-3.5-turbo
- Max Tokens: 400
- Temperature: 0.7

System Message:
"You are an experienced ecommerce entrepreneur who helps others in Reddit communities. Generate authentic, helpful responses that provide real value first. Sound natural and personalized, not like marketing copy. Match the community's communication style."

User Message:
"Generate an authentic Reddit response for this situation:

ORIGINAL POST:
Title: {{3.title}}
Content: {{3.content}}
Subreddit: {{3.subreddit}}

ANALYSIS:
Strategy: {{4.strategy}}
Template Category: {{4.template_category}}
Pain Point: {{4.pain_point}}
User Context: {{4.user_context}}
Competitor Mentioned: {{4.competitor_mentioned}}
Priority: {{1.priority}}

REQUIREMENTS:
- Sound like a real person, not a marketer
- Provide genuine value first
- Match {{3.subreddit}} communication style
- Keep response 150-350 words
- Include Loox mention only if strategy allows and feels natural
- End with a question to encourage engagement
- Use {{4.subreddit_style}} tone

Strategy-specific guidelines:
- direct_recommendation: Can mention Loox as one of several options
- helpful_positioning: Lead with advice, mention Loox in context
- balanced_comparison: Include Loox in balanced comparison
- educational: Focus on education, minimal tool mentions

Generate the response:"
```

#### **6. JavaScript Module: Quality Validation**
```javascript
// Validate response quality and authenticity
const response = `{{5.choices[0].message.content}}`;
const originalPost = {
    title: '{{3.title}}',
    subreddit: '{{3.subreddit}}'
};

function validateResponse(response, originalPost) {
    const checks = {
        // Length check
        length_appropriate: response.length >= 150 && response.length <= 600,
        
        // Natural language patterns
        sounds_human: !containsMarketingSpeak(response),
        
        // Value-first principle
        provides_value: hasEducationalContent(response),
        
        // Community fit  
        matches_subreddit: matchesCommunityStyle(response, originalPost.subreddit),
        
        // Subtle positioning
        not_salesy: !isOvertlySales(response)
    };
    
    function containsMarketingSpeak(text) {
        const marketingPhrases = [
            'our product', 'industry leader', 'best solution',
            'revolutionary', 'game changer', 'cutting edge',
            'contact us', 'sign up now', 'limited time'
        ];
        return marketingPhrases.some(phrase => 
            text.toLowerCase().includes(phrase)
        );
    }
    
    function hasEducationalContent(text) {
        const educationalIndicators = [
            'here\'s what', 'in my experience', 'what i learned',
            'the key is', 'important to', 'consider',
            'depends on', 'look for', 'make sure'
        ];
        return educationalIndicators.some(indicator =>
            text.toLowerCase().includes(indicator)
        );
    }
    
    function matchesCommunityStyle(text, subreddit) {
        // Basic subreddit style matching
        if (subreddit === 'r/entrepreneur') {
            return text.includes('experience') || text.includes('learned');
        }
        if (subreddit === 'r/shopify') {
            return text.includes('store') || text.includes('conversion');
        }
        return true; // Default pass
    }
    
    function isOvertlySales(text) {
        const salesyPhrases = [
            'best choice', 'you should definitely',
            'highly recommend', 'perfect solution',
            'exactly what you need'
        ];
        const salesyCount = salesyPhrases.filter(phrase =>
            text.toLowerCase().includes(phrase)
        ).length;
        return salesyCount >= 2;
    }
    
    const passedChecks = Object.values(checks).filter(Boolean).length;
    const totalChecks = Object.keys(checks).length;
    
    return {
        authenticity_score: Math.round((passedChecks / totalChecks) * 10 * 10) / 10,
        failed_checks: Object.keys(checks).filter(key => !checks[key]),
        approved: passedChecks >= totalChecks * 0.8
    };
}

const validation = validateResponse(response, originalPost);

return {
    response_text: response,
    authenticity_score: validation.authenticity_score,
    failed_checks: validation.failed_checks.join(', '),
    auto_approved: validation.approved,
    requires_human_review: !validation.approved || {{1.priority}} === 'High',
    generated_date: new Date().toISOString()
};
```

#### **7. Filter Module: Quality Gate**
```
Condition:
{{6.authenticity_score}} Greater than or equal to 7.0

Continue only if quality threshold met
```

#### **8. Google Sheets: Add Row (Response_Queue)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Response_Queue
- Values:
  Row 1: {{1.post_id}}_{{6.generated_date}} (Queue_ID)
  Row 2: {{1.post_id}}
  Row 3: {{6.response_text}}
  Row 4: {{4.strategy}}
  Row 5: {{4.template_category}}
  Row 6: {{6.authenticity_score}}
  Row 7: (Account assignment - handled in Scenario 4)
  Row 8: (Scheduled time - handled in Scenario 4)
  Row 9: {{if(6.requires_human_review, "Pending_Review", "Approved")}}
  Row 10: {{6.requires_human_review}}
  Row 11: (Review notes - for human reviewer)
  Row 12: {{6.generated_date}}
  Row 13: (Approved date - when human approves)
  Row 14: (Manual edits - any changes made)
```

#### **9. Google Sheets: Update Row (Scored_Posts)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Scored_Posts
- Row Number: {{1.__row_number__}}
- Column N (Status): Response_Generated
```

---

## Make.com Scenario 4: Account Manager & Poster

**Purpose:** Assign accounts, schedule posting, and execute responses
**Trigger:** Google Sheets watch rows (Response_Queue)
**Runtime:** ~1-2 minutes initial, then scheduled posting

### **Module Flow:**

#### **1. Google Sheets: Watch Rows (Response_Queue)**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Response_Queue
- Trigger Column: Status
- Trigger Value: Approved
- Limit: 1
```

#### **2. Filter Module: Posting Hours Check**
```
Condition:
Current time between 9:00 AM and 7:00 PM (your timezone)

Skip posting outside business hours
```

#### **3. Google Sheets: Get Values (Multiple Sheets)**
```
Settings for Account Data:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Account_Health
- Range: A:N
- Return: All Rows

Settings for Post Data:
- Spreadsheet: Loox Reddit Monitor  
- Sheet: Raw_Posts
- Range: A:M
- Filter: Column A (Post_ID) = {{1.post_id}}
- Return: Single Row
```

#### **4. JavaScript Module: Account Selection & Timing**
```javascript
// Smart account selection based on health, history, and cooldowns
const queueData = {
    post_id: '{{1.post_id}}',
    priority: '{{1.priority}}', // from scored_posts lookup
    subreddit: '{{4.subreddit}}' // from raw_posts lookup
};

const accounts = {{3.accounts}}; // Array from Account_Health sheet

function selectOptimalAccount(queueData, accounts) {
    const now = new Date();
    
    // Filter available accounts
    const availableAccounts = accounts.filter(account => {
        const cooldownUntil = new Date(account.cooldown_until || 0);
        const postsToday = parseInt(account.posts_today || 0);
        const maxDaily = getMaxDaily(account.account_type);
        
        return (
            account.status === 'Active' &&
            account.health_score >= 7 &&
            cooldownUntil <= now &&
            postsToday < maxDaily
        );
    });
    
    if (availableAccounts.length === 0) {
        return null; // No accounts available
    }
    
    // Scoring function for account selection
    function scoreAccount(account) {
        let score = 0;
        
        // Health score (0-50 points)
        score += account.health_score * 5;
        
        // Subreddit experience (0-20 points)
        const subredditKarma = getSubredditKarma(account, queueData.subreddit);
        score += Math.min(subredditKarma / 10, 20);
        
        // Recency penalty (avoid recently used accounts)
        const daysSinceLastPost = getDaysSince(account.last_post_date);
        score += Math.min(daysSinceLastPost * 2, 10);
        
        // Account type preference for priority
        if (queueData.priority === 'High' && account.account_type === 'Primary') {
            score += 15;
        } else if (queueData.priority === 'Medium' && account.account_type === 'Secondary') {
            score += 10;
        }
        
        return score;
    }
    
    // Select best account
    const scoredAccounts = availableAccounts.map(account => ({
        ...account,
        selection_score: scoreAccount(account)
    }));
    
    return scoredAccounts.sort((a, b) => b.selection_score - a.selection_score)[0];
}

function calculateOptimalPostTime(account, priority) {
    // Base delay: 2-6 hours for natural timing
    let baseDelayHours = Math.random() * 4 + 2;
    
    // Adjust for priority
    if (priority === 'High') {
        baseDelayHours *= 0.7; // Post sooner for high priority
    }
    
    // Adjust for account cooldown requirements
    const accountCooldown = getAccountCooldown(account.account_type);
    const timeSinceLastPost = getHoursSince(account.last_post_date);
    
    if (timeSinceLastPost < accountCooldown) {
        baseDelayHours = Math.max(baseDelayHours, accountCooldown - timeSinceLastPost);
    }
    
    return baseDelayHours;
}

function getMaxDaily(accountType) {
    const limits = { 'Primary': 1, 'Secondary': 2, 'Support': 3 };
    return limits[accountType] || 1;
}

function getAccountCooldown(accountType) {
    const cooldowns = { 'Primary': 48, 'Secondary': 24, 'Support': 12 };
    return cooldowns[accountType] || 24;
}

function getSubredditKarma(account, subreddit) {
    try {
        const karmaData = JSON.parse(account.subreddit_karma || '{}');
        return karmaData[subreddit] || 0;
    } catch (e) {
        return 0;
    }
}

function getDaysSince(dateString) {
    if (!dateString) return 999;
    const then = new Date(dateString);
    const now = new Date();
    return (now - then) / (1000 * 60 * 60 * 24);
}

function getHoursSince(dateString) {
    if (!dateString) return 999;
    const then = new Date(dateString);
    const now = new Date();
    return (now - then) / (1000 * 60 * 60);
}

// Execute selection
const selectedAccount = selectOptimalAccount(queueData, accounts);

if (!selectedAccount) {
    return {
        error: 'No suitable accounts available',
        retry_in_hours: 2
    };
}

const delayHours = calculateOptimalPostTime(selectedAccount, queueData.priority);
const scheduledTime = new Date(Date.now() + delayHours * 60 * 60 * 1000);

return {
    selected_account: selectedAccount.account_name,
    account_type: selectedAccount.account_type,
    delay_hours: delayHours,
    scheduled_time: scheduledTime.toISOString(),
    selection_reason: `Health: ${selectedAccount.health_score}, Score: ${selectedAccount.selection_score}`
};
```

#### **5. Router Module**
```
Route 1: Account Available (has selected_account)
Route 2: No Account Available (error condition)
```

#### **5A. Delay Module** 
```
Settings:
- Delay: {{4.delay_hours}} hours
- Unit: Hours
```

#### **6. Google Sheets: Update Row (Response_Queue) - Assignment**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Response_Queue
- Row Number: {{1.__row_number__}}
- Column G (Assigned_Account): {{4.selected_account}}
- Column H (Scheduled_Time): {{4.scheduled_time}}
- Column I (Status): Scheduled
```

#### **7. HTTP Module: Post to Reddit**
```
Settings:
- URL: https://oauth.reddit.com/api/comment
- Method: POST
- Headers:
  - Authorization: Bearer {{getAccountToken(4.selected_account)}}
  - User-Agent: {{getAccountUserAgent(4.selected_account)}}
  - Content-Type: application/x-www-form-urlencoded

Body (form-urlencoded):
- api_type: json
- thing_id: {{1.post_id}}
- text: {{1.response_text}}

Note: You'll need to implement Reddit OAuth flow for each account
```

#### **8. JavaScript Module: Process Reddit Response**
```javascript
// Process the Reddit API response
const redditResponse = {{7}};

if (redditResponse.json && redditResponse.json.errors.length === 0) {
    // Successful post
    const commentData = redditResponse.json.data.things[0].data;
    
    return {
        success: true,
        comment_id: commentData.name,
        comment_url: 'https://reddit.com' + commentData.permalink,
        posted_at: new Date().toISOString(),
        initial_score: commentData.score || 0
    };
} else {
    // Failed post
    const errors = redditResponse.json ? redditResponse.json.errors : ['Unknown error'];
    
    return {
        success: false,
        error: errors.join(', '),
        requires_manual_review: true
    };
}
```

#### **9A. Google Sheets: Add Row (Posted_Responses) - Success Path**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Posted_Responses
- Values:
  Row 1: {{8.comment_id}} (Response_ID)
  Row 2: {{1.post_id}}
  Row 3: {{8.comment_id}}
  Row 4: {{4.selected_account}}
  Row 5: {{4.subreddit}}
  Row 6: {{8.posted_at}}
  Row 7: {{1.response_text}}
  Row 8: {{8.initial_score}}
  Row 9: (Final_Upvotes - updated later)
  Row 10: 0 (Replies_Received - updated later)
  Row 11: (Engagement_Score - calculated later)
  Row 12: Pending (Community_Reaction - assessed later)
  Row 13: FALSE (Conversion_Tracked)
  Row 14: Posted successfully via automation
```

#### **10. Google Sheets: Update Account Health**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Account_Health
- Find Row: Column A (Account_Name) = {{4.selected_account}}
- Updates:
  - Column F (Last_Post_Date): {{8.posted_at}}
  - Column G (Posts_Today): {{increment current value}}
  - Column H (Posts_This_Week): {{increment current value}}
  - Column K (Cooldown_Until): {{calculate based on account type}}
  - Column M (Last_Updated): {{current timestamp}}
```

#### **11. Google Sheets: Update Row (Response_Queue) - Final Status**
```
Settings:
- Spreadsheet: Loox Reddit Monitor
- Sheet: Response_Queue
- Row Number: {{1.__row_number__}}
- Column I (Status): Posted
```

---

## Daily Maintenance & Monitoring

### **Additional Make.com Scenarios for System Health:**

#### **Scenario 5: Daily Health Check**
**Purpose:** Monitor account health and system performance
**Trigger:** Daily at 6:00 AM

```
Modules:
1. Schedule Trigger (Daily 6 AM)
2. Reddit API: Check account status for each account
3. Calculate health scores
4. Update Account_Health sheet
5. Generate daily report
6. Send alerts if issues detected
```

#### **Scenario 6: Engagement Tracker**
**Purpose:** Track engagement on posted responses
**Trigger:** Every 6 hours

```
Modules:
1. Schedule Trigger (Every 6 hours)
2. Google Sheets: Get Posted_Responses with pending engagement
3. Reddit API: Get comment data for each response
4. Update engagement metrics
5. Calculate community reaction
6. Update Posted_Responses sheet
```

---

## Setup Instructions

### **1. Google Sheets Setup**
1. Create new Google Sheets file named "Loox Reddit Monitor"
2. Add 5 sheets with exact column structures above
3. Share with Make.com service account
4. Note the spreadsheet ID for Make.com

### **2. Reddit API Setup**
1. Create Reddit application at reddit.com/prefs/apps
2. Get client ID, client secret
3. Implement OAuth flow for each account
4. Store tokens securely in Make.com variables

### **3. OpenAI API Setup**
1. Get OpenAI API key
2. Add to Make.com connections
3. Monitor usage and set budgets

### **4. Make.com Configuration**
1. Create 4 scenarios as detailed above
2. Set up error handling and notifications
3. Test each scenario with sample data
4. Enable scenarios in sequence

### **5. Account Preparation**
Start warming accounts immediately (30-60 day process):
- Create 8 Reddit accounts with diverse backgrounds
- Build karma through authentic participation
- Establish subreddit presence gradually
- Document account details in Account_Health sheet

---

## Error Handling & Monitoring

### **Common Error Scenarios:**
- Reddit API rate limits → Implement exponential backoff
- OpenAI API failures → Retry logic with fallback scoring
- Google Sheets quota limits → Optimize read/write operations
- Account health issues → Automatic pause and recovery

### **Monitoring Setup:**
- Daily health reports via email
- Real-time alerts for critical failures  
- Weekly performance summaries
- Monthly optimization reviews

This implementation guide provides complete technical specifications for building your Reddit monitoring system with Make.com and Google Sheets. Follow the setup instructions and test each scenario thoroughly before going live.

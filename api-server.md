# VozLine Sports Data API — Vapi Function Server
# GoStar Digital LLC, Puerto Rico
# 
# This server handles function calls from the Vapi agent
# and returns real-time sports data.
#
# DEPLOYMENT: Cloudflare Workers (recommended) or any serverless platform
# NO RENDER. NO NODE SERVER. Cloudflare Workers only.

# ============================================================
# OPTION 1: USE VAPI'S BUILT-IN SPORTS TOOL
# ============================================================
# 
# Vapi has a built-in sports data tool! You can skip building
# a custom server entirely by using their native integration.
#
# In Vapi Dashboard → Agent → Tools → Add Tool:
# Select "Sports Data" from the available tools
#
# This gives you:
# - Live scores (NBA, NFL, NHL, MLB, MLS, EPL, etc.)
# - Standings
# - Game stats
# - Player stats
#
# The functions in the system prompt (get_sports_scores, 
# get_standings, get_game_stats) will automatically route
# to Vapi's sports data provider.
#
# ============================================================


# ============================================================
# OPTION 2: CUSTOM CLOUDFLARE WORKER (if you need more control)
# ============================================================

# File: worker.js (Cloudflare Workers)

```javascript
// VozLine Sports API — Cloudflare Worker
// Handles Vapi function calls for live sports data

const SPORTS_API_KEY = "YOUR_API_SPORTS_KEY"; // Get from api-sports.io
const SPORTS_API_HOST = "v3.football.api-sports.io";

// League ID mappings for API-Sports
const LEAGUE_IDS = {
  // Football/Soccer
  epl: 39,        // English Premier League
  la_liga: 140,   // La Liga
  serie_a: 135,   // Serie A
  bundesliga: 78, // Bundesliga
  ligue_1: 61,    // Ligue 1
  mls: 253,       // MLS
  champions_league: 2, // Champions League
  
  // For NBA, NFL, MLB, NHL — use different API endpoints
  // API-Sports has separate APIs for each sport
};

export default {
  async fetch(request, env) {
    // Handle CORS
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": "POST, OPTIONS",
          "Access-Control-Allow-Headers": "Content-Type",
        },
      });
    }

    if (request.method !== "POST") {
      return new Response("Method not allowed", { status: 405 });
    }

    try {
      const body = await request.json();
      const { message } = body;
      
      // Vapi sends function calls in message.tool_calls
      if (message?.tool_calls) {
        const results = [];
        
        for (const toolCall of message.tool_calls) {
          const { name, arguments: args } = toolCall.function;
          let result;
          
          switch (name) {
            case "get_sports_scores":
              result = await getScores(args.league, args.team);
              break;
            case "get_standings":
              result = await getStandings(args.league, args.conference);
              break;
            case "get_game_stats":
              result = await getGameStats(args.league, args.game_id);
              break;
            default:
              result = { error: "Unknown function" };
          }
          
          results.push({
            tool_call_id: toolCall.id,
            content: JSON.stringify(result),
          });
        }
        
        return new Response(JSON.stringify({ results }), {
          headers: {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
          },
        });
      }

      return new Response(JSON.stringify({ error: "No tool calls found" }), {
        status: 400,
        headers: { "Content-Type": "application/json" },
      });
      
    } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), {
        status: 500,
        headers: { "Content-Type": "application/json" },
      });
    }
  },
};

// ============================================================
// SPORTS DATA FUNCTIONS
// ============================================================

async function getScores(league, team) {
  // For soccer leagues, use API-Sports Football
  if (["epl", "la_liga", "serie_a", "bundesliga", "ligue_1", "mls", "champions_league"].includes(league)) {
    return await getFootballScores(league, team);
  }
  
  // For American sports, use different API
  // Options: SportRadar, ESPN unofficial, balldontlie (NBA), etc.
  if (["nba", "wnba", "nfl", "nhl", "mlb", "ncaamb", "ncaafb"].includes(league)) {
    return await getAmericanSportsScores(league, team);
  }
  
  return { message: "League not supported yet", league };
}

async function getFootballScores(league, team) {
  const leagueId = LEAGUE_IDS[league];
  const today = new Date().toISOString().split("T")[0];
  
  const url = `https://${SPORTS_API_HOST}/fixtures?league=${leagueId}&season=2024&from=${today}&to=${today}`;
  
  const response = await fetch(url, {
    headers: {
      "x-rapidapi-key": SPORTS_API_KEY,
      "x-rapidapi-host": SPORTS_API_HOST,
    },
  });
  
  const data = await response.json();
  
  if (data.response && data.response.length > 0) {
    const games = data.response.map(match => ({
      home: match.teams.home.name,
      away: match.teams.away.name,
      score: `${match.goals.home ?? 0} - ${match.goals.away ?? 0}`,
      status: match.fixture.status.long,
      time: match.fixture.date,
      venue: match.fixture.venue.name,
    }));
    
    // Filter by team if specified
    if (team) {
      const filtered = games.filter(g => 
        g.home.toLowerCase().includes(team.toLowerCase()) ||
        g.away.toLowerCase().includes(team.toLowerCase())
      );
      return { games: filtered, league, query: team };
    }
    
    return { games, league };
  }
  
  return { message: "No games found for today", league };
}

async function getAmericanSportsScores(league, team) {
  // Using balldontlie for NBA (free, no key required)
  if (league === "nba") {
    const today = new Date().toISOString().split("T")[0];
    const url = `https://api.balldontlie.io/v1/games?dates[]=${today}`;
    
    const response = await fetch(url, {
      headers: {
        "Authorization": "YOUR_BALLDONTLIE_KEY" // Free tier available
      }
    });
    
    const data = await response.json();
    
    if (data.data && data.data.length > 0) {
      const games = data.data.map(game => ({
        home: game.home_team.full_name,
        away: game.visitor_team.full_name,
        homeScore: game.home_team_score,
        awayScore: game.visitor_team_score,
        status: game.status,
        time: game.time || "Final",
      }));
      
      if (team) {
        const filtered = games.filter(g =>
          g.home.toLowerCase().includes(team.toLowerCase()) ||
          g.away.toLowerCase().includes(team.toLowerCase())
        );
        return { games: filtered, league: "NBA", query: team };
      }
      
      return { games, league: "NBA" };
    }
    
    return { message: "No NBA games found for today" };
  }
  
  // For other American sports, return placeholder
  // You'd integrate SportRadar or similar here
  return {
    message: `Live ${league.toUpperCase()} data available with SportRadar integration`,
    demo: true,
    league: league.toUpperCase(),
  };
}

async function getStandings(league, conference) {
  // NBA standings via balldontlie
  if (league === "nba") {
    // balldontlie doesn't have standings, so we'd use SportRadar
    // For demo, return structure
    return {
      message: "NBA standings available with full integration",
      sample: [
        { rank: 1, team: "Boston Celtics", wins: 45, losses: 12, pct: ".789" },
        { rank: 2, team: "Cleveland Cavaliers", wins: 43, losses: 15, pct: ".741" },
        { rank: 3, team: "New York Knicks", wins: 38, losses: 20, pct: ".655" },
      ],
      conference: conference || "east",
    };
  }
  
  // Soccer standings via API-Sports
  if (LEAGUE_IDS[league]) {
    const leagueId = LEAGUE_IDS[league];
    const url = `https://${SPORTS_API_HOST}/standings?league=${leagueId}&season=2024`;
    
    const response = await fetch(url, {
      headers: {
        "x-rapidapi-key": SPORTS_API_KEY,
        "x-rapidapi-host": SPORTS_API_HOST,
      },
    });
    
    const data = await response.json();
    
    if (data.response?.[0]?.league?.standings?.[0]) {
      const standings = data.response[0].league.standings[0].slice(0, 10).map(team => ({
        rank: team.rank,
        team: team.team.name,
        played: team.all.played,
        wins: team.all.win,
        draws: team.all.draw,
        losses: team.all.lose,
        points: team.points,
        goalDiff: team.goalsDiff,
      }));
      
      return { standings, league };
    }
  }
  
  return { message: "Standings not available", league };
}

async function getGameStats(league, gameId) {
  // Detailed game stats require specific game ID
  // This would come from a previous scores call
  
  if (league === "nba" && gameId) {
    const url = `https://api.balldontlie.io/v1/stats?game_ids[]=${gameId}`;
    
    const response = await fetch(url, {
      headers: {
        "Authorization": "YOUR_BALLDONTLIE_KEY"
      }
    });
    
    const data = await response.json();
    
    if (data.data && data.data.length > 0) {
      // Get top performers
      const sorted = data.data.sort((a, b) => b.pts - a.pts);
      const topScorer = sorted[0];
      
      return {
        gameId,
        topScorer: {
          player: `${topScorer.player.first_name} ${topScorer.player.last_name}`,
          team: topScorer.team.full_name,
          points: topScorer.pts,
          rebounds: topScorer.reb,
          assists: topScorer.ast,
        },
        allStats: sorted.slice(0, 5).map(p => ({
          player: `${p.player.first_name} ${p.player.last_name}`,
          pts: p.pts,
          reb: p.reb,
          ast: p.ast,
          team: p.team.abbreviation,
        })),
      };
    }
  }
  
  return { message: "Game stats require valid game ID from scores", gameId };
}
```

# ============================================================
# VAPI WEBHOOK CONFIGURATION
# ============================================================

# In Vapi Dashboard:
# 1. Go to your agent
# 2. Click "Tools" 
# 3. Add Custom Tool
# 4. Set Server URL to your Cloudflare Worker URL
# 5. The worker handles all three functions

# Alternatively, use Vapi's built-in sports tool (RECOMMENDED):
# - No server needed
# - Already integrated with sports data providers
# - Just enable it in the Tools section


# ============================================================
# API KEYS NEEDED
# ============================================================

# 1. API-Sports (for soccer/football)
#    - Sign up: https://api-sports.io/
#    - Free tier: 100 requests/day
#    - Paid: starts at $15/month for 7,500 requests
#
# 2. balldontlie (for NBA)
#    - Sign up: https://www.balldontlie.io/
#    - Free tier: 30 requests/minute
#    - Paid: starts at $9.99/month
#
# 3. SportRadar (for all sports - premium)
#    - Sign up: https://sportradar.com/
#    - Trial available
#    - Full coverage of all leagues
#    - This is what ESPN and major apps use
#
# RECOMMENDED FOR DEMO:
# - Use Vapi's built-in sports tool (free with Vapi)
# - OR balldontlie for NBA (free) + API-Sports free tier for soccer


# ============================================================
# DEPLOYMENT TO CLOUDFLARE WORKERS
# ============================================================

# 1. Install Wrangler CLI:
#    npm install -g wrangler
#
# 2. Login to Cloudflare:
#    wrangler login
#
# 3. Create worker:
#    wrangler init vozline-sports-api
#
# 4. Copy the worker.js code above into src/index.js
#
# 5. Add API keys as secrets:
#    wrangler secret put SPORTS_API_KEY
#    wrangler secret put BALLDONTLIE_KEY
#
# 6. Deploy:
#    wrangler deploy
#
# 7. Copy the worker URL and add to Vapi agent tools config


# ============================================================
# TESTING
# ============================================================

# Test the worker locally:
# wrangler dev
#
# Send test request:
# curl -X POST http://localhost:8787 \
#   -H "Content-Type: application/json" \
#   -d '{
#     "message": {
#       "tool_calls": [{
#         "id": "test123",
#         "function": {
#           "name": "get_sports_scores",
#           "arguments": "{\"league\": \"nba\", \"team\": \"lakers\"}"
#         }
#       }]
#     }
#   }'

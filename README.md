# Rate Limiting
I chose to use 10 per minute and 50 per day, since once every 6 seconds is far too many, and once every 30 minutes also seems like too many uses. 


$ for i in $(seq 1 12); do   curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/submit     -H "Content-Type: application/json"     -d '{"text": "This is a test submission for rate limit testing purposes only.", "creator_id": "ratelimit-test"}'; done
200
200
200
200
200
200
200
200
200
200
429
429
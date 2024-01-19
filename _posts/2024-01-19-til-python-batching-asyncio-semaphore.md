---
layout: post
title: "TIL: Asyncio for API Requests - Batching vs. Semaphore"
description: "This article describes the difference between manual batching vs Semaphore."
---

In the world of API requests, like those to Wikipedia, we often face the
challenge of rate limits. To navigate this efficiently with Python's asyncio, I
dived into two strategies: **Batching** and **Semaphore**. Batching breaks down
requests into manageable groups, processing them sequentially like a relay
race. On the flip side, Semaphore manages how many requests run concurrently,
akin to a bouncer controlling a nightclub's entry.

During my tests on a Wikipedia public API to retrieve data for 100 entities, I uncovered some intriguing insights. With batching, I sent 10 requests at a time in systematic bursts, ensuring a steady and safer approach. Semaphore, though set to 10 concurrent requests, initially stumbled upon rate limits, fetching fewer results. This mimicked real-world API interactions, like Wikipedia's defense mechanisms against potential abuse.

The key turning point was introducing a delay in the Semaphore method. This small tweak made a significant difference, balancing speed and adherence to API rate limits. It became clear that a more measured approach could outshine sheer speed, particularly when external constraints like API limitations come into play.

### Key Lessons Learned:

**The Reality of Rate Limiting**: APIs have built-in measures to thwart abuse. Bombarding an API too rapidly can result in blocks. It's crucial to verify the actual results, ensuring they are valid and not empty (None).

**The Safety of Batching**: Batching naturally spaces out requests, effectively reducing the risk of hitting rate limits. While its simplicity might seem less 'Pythonic' to some, its straightforward nature (processing batch by batch) is often more reliable in avoiding rate limit triggers.

**The Nuance of Semaphores*k: Semaphores, while appearing more Pythonic, require careful calibration. Not all results are guaranteed to be complete. Adjusting the concurrency limits and strategically introducing delays can prevent triggering the API's defensive measures.


#### Below is the test script:
{% highlight python %}
import asyncio
import time
from typing import Dict, List, Optional

import aiohttp


# Async function to fetch data from Wikipedia API
async def fetch_wikipedia_page(
    session: aiohttp.ClientSession, entity: str
) -> Optional[Dict[str, str]]:
    url = "https://en.wikipedia.org/w/rest.php/v1/search/page"
    params = {
        "limit": 1,
        "q": entity,
    }
    async with session.get(url, params=params) as response:
        if response.status == 200:
            return await response.json()
        return None


# Batching method
async def fetch_with_batching(
    entities: List[str], batch_size=10
) -> List[Optional[Dict[str, str]]]:
    async with aiohttp.ClientSession() as session:
        results = []
        for i in range(0, len(entities), batch_size):
            batch = entities[i : i + batch_size]
            tasks = [fetch_wikipedia_page(session, entity) for entity in batch]
            results += await asyncio.gather(*tasks)
        return results


# Semaphore method
async def fetch_with_semaphore(
    entities: List[str], max_concurrent_requests=10
) -> List[Optional[Dict[str, str]]]:
    async with aiohttp.ClientSession() as session:
        semaphore = asyncio.Semaphore(max_concurrent_requests)
        tasks = [
            fetch_with_semaphore_task(session, semaphore, entity) for entity in entities
        ]
        return await asyncio.gather(*tasks)


async def fetch_with_semaphore_task(
    session: aiohttp.ClientSession, semaphore: asyncio.Semaphore, entity: str, delay=0.1
) -> Optional[Dict[str, str]]:
    async with semaphore:
        await asyncio.sleep(delay)
        return await fetch_wikipedia_page(session, entity)



# Test function to compare both methods
async def compare_methods(entities: List[str]):
    # Testing batching method
    start_time = time.time()
    results_batching = await fetch_with_batching(entities, batch_size=10)
    end_time = time.time()
    print(f"Batching Method Time: {end_time - start_time:.4f} seconds")

    # Testing semaphore method
    start_time = time.time()
    results_semaphore = await fetch_with_semaphore(entities, max_concurrent_requests=10)
    end_time = time.time()
    print(f"Semaphore Method Time: {end_time - start_time:.4f} seconds")

    return results_batching, results_semaphore


# List of entities for testing
entities = ["Python", "JavaScript"] * 50
print(f"DEBUG: {len(entities) = }")

# Run the comparison
batching_results, semaphore_results = asyncio.run(compare_methods(entities))

non_none_batching_results = [item for item in batching_results if item]
print(f"DEBUG: {len(non_none_batching_results) = }")

non_none_semaphore_results = [item for item in semaphore_results if item]
print(f"DEBUG: {len(non_none_semaphore_results) = }")

"""
>>> DEBUG: len(entities) = 100
>>> Batching Method Time: 3.6377 seconds
>>> Semaphore Method Time: 2.4993 seconds
>>> DEBUG: len(non_none_batching_results) = 100
>>> DEBUG: len(non_none_semaphore_results) = 45
"""
{% endhighlight %}

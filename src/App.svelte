<script>
  const TOKEN = 'Token ESV_API_TOKEN';
  const BOOKS = [
    'Genesis',
    'Exodus',
    'Leviticus',
    'Numbers',
    'Deuteronomy',
    'Joshua',
    'Judges',
    'Ruth',
    '1 Samuel',
    '2 Samuel',
    '1 Kings',
    '2 Kings',
    '1 Chronicles',
    '2 Chronicles',
    'Ezra',
    'Nehemiah',
    'Esther',
    'Job',
    'Psalm',
    'Proverbs',
    'Ecclesiastes',
    'Song of Songs',
    'Isaiah',
    'Jeremiah',
    'Lamentations',
    'Ezekiel',
    'Daniel',
    'Hosea',
    'Joel',
    'Amos',
    'Obadiah',
    'Jonah',
    'Micah',
    'Nahum',
    'Habakkuk',
    'Zephaniah',
    'Haggai',
    'Zechariah',
    'Malachi',
    'Matthew',
    'Mark',
    'Luke',
    'John',
    'Acts',
    'Romans',
    '1 Corinthians',
    '2 Corinthians',
    'Galatians',
    'Ephesians',
    'Philippians',
    'Colossians',
    '1 Thessalonians',
    '2 Thessalonians',
    '1 Timothy',
    '2 Timothy',
    'Titus',
    'Philemon',
    'Hebrews',
    'James',
    '1 Peter',
    '2 Peter',
    '1 John',
    '2 John',
    '3 John',
    'Jude',
    'Revelation'
  ];
  let markers = [];
  let book;
  let startDay = new Date().toISOString().substring(0, 10);
  const queryBook = async (book) => {
    let endOfBookReached = false;
    let currentChapter = 1;
    let currentVerse = 1;
    const output = [];
    while (!endOfBookReached) {
      const data = await fetch(`https://api.esv.org/v3/passage/html/?q=${book.replace(' ', '+')}+${currentChapter}:&include-passage-references=false&include-first-verse-numbers=false&include-footnotes=false&include-footnote-body=false&include-short-copyright=false`, { headers: { Authorization: TOKEN, }, })
      const json = await data.json();
      output.push(json.passages[0]);
      const nextBook = parseInt(json.passage_meta[0].next_chapter[0].toString().substring(0, 2));
      const currentBook = BOOKS.indexOf(book) + 1;
      if (nextBook > currentBook) {
        endOfBookReached = true;
      } else {
        currentChapter++;
      }
    }
    const verseText = output.join('');
    const headingsSeparated = verseText.split('<h3 ').filter(s => s !== '').map(s => `<h3 ${s}`);
    const versesSeparated = headingsSeparated.flatMap(section => {
      const allSections = section.split('<b ').map((s, i) => i === 0 ? s : `<b ${s}`);
      return [allSections[0] + allSections[1], ...allSections.slice(2, allSections.length)];
    })
    markers.push(versesSeparated.length);
    return versesSeparated.map((verse, index) => ({ verse, globalIndex: index }));
  };
  let versesPromise = Promise.resolve([]);
  let verseChunks = [];
  $: calculateVerseChunks(markers, versesPromise, startDay);
  const go = (book) => {
    versesPromise = queryBook(book);
    markers = [];
  }
  const calculateVerseChunks = (markers, versesPromise, startDay) => {
    versesPromise.then((verses) => {
      const newVerseChunks = [];
      let currentIndex = 0;
      let currentDay = new Date(startDay);
      markers.forEach((index) => {
        const slicedVerses = verses.slice(currentIndex, index);
        const firstVerseMatches = slicedVerses[0].verse.match(/<b .*?id="\w\d{2}(\d{3})(\d{3})/);
        const lastVerseMatches = slicedVerses[slicedVerses.length - 1].verse.match(/<b .*?id="\w\d{2}(\d{3})(\d{3})/);
        newVerseChunks.push({
          verses: slicedVerses,
          endIndex: index,
          day: currentDay.toISOString().substring(0, 10),
          firstVerse: firstVerseMatches && `${parseInt(firstVerseMatches[1])}:${parseInt(firstVerseMatches[2])}`,
          lastVerse: lastVerseMatches && `${parseInt(lastVerseMatches[1])}:${parseInt(lastVerseMatches[2])}`
        });
        currentDay = new Date(currentDay.getTime() + 60 * 60 * 24 * 1000)
        currentIndex = index;
      });
      verseChunks = newVerseChunks;
    })
  }
  const toggleMarker = (editIndex) => {
    if (markers.includes(editIndex)) {
      markers.splice(markers.indexOf(editIndex), 1);
    } else {
      markers.push(editIndex)
    }
    markers = markers.sort((a, b) => a-b);
  }
  const wordCount = (verses) => {
    const div = document.createElement('div');
    div.innerHTML = verses.map(v => v.verse).join(' ');
    div.querySelectorAll('b').forEach(e => e.parentNode.removeChild(e));
    div.querySelectorAll('h3').forEach(e => e.parentNode.removeChild(e));
    return div.textContent.split(/\s/).filter(w => w.trim() !== '').length;
  }
</script>

<style>
  :global(h3) {
    display: inline-block;
  }
  .divider {
    font-size: 1.5rem;
    cursor: pointer;
  }
  .selected {
    color: red;
  }
</style>

<input bind:value={book} /><button on:click={() => go(book)}>Go</button>
<input bind:value={startDay} type="date" />
<br />
{#await versesPromise then _}
  {#each verseChunks as { verses, endIndex, day, firstVerse, lastVerse } (endIndex.toString())}
    {#each verses as { verse, globalIndex }, index (globalIndex.toString())}
      {#if index > 0}
        <span on:click={() => toggleMarker(globalIndex)} class="divider">|</span>
      {/if}
      {@html verse}
    {/each}
    <br />
    <br />
    <button on:click={() => toggleMarker(endIndex)}>Remove Divider</button>
    <hr />
    Word Count: {wordCount(verses)}
    Day: {day}
    Section: {book} {firstVerse} - {lastVerse}
    <br />
    <br />
  {/each}

  <h1>Schedule</h1>
  {#each verseChunks as { verses, endIndex, day, firstVerse, lastVerse } (endIndex.toString())}
    <p>Read {book} {firstVerse} - {lastVerse} on {day}</p>
  {/each}
{/await}

# RFC: GraphQL Composite Schemas

There's a _lot_ of different ways of combining multiple GraphQL schemas together to form a larger composite schema, and the wide variety of options can make it challenging to design schemas in a way that will be easy to compose later. To name but a few approaches:

- Schema merging and stitching (various implementations in many languages)
- GraphQL modules and other "extend type"-based approaches
- Federation (Apollo's v1 and v2, Mercurius, WunderGraph, Hot Chocolate, etc)
- Hasura's GraphQL Joins
- Atlassian's Nadel
- StepZen

Though these are all separate solutions to similar problems, there are various concerns that most of them have to consider:

- Identifying the schemas to join
- Discovering/defining the relationships between the types/schemas
- Schema manipulation to avoid conflicts (e.g. renaming types/fields)
- Resolving questions around type "ownership", redundancy and/or sharing; detecting conflicts
- Detecting breaking changes
- Actually fetching the relevant data, and combining it to fulfil the GraphQL request
- Managing composition metadata with respect to handling the above concerns
- etc

It feels like it would be of benefit to the ecosystem at large if there were a shared specification that covers a few of these needs.

If this is of interest to you, please enter your name, GitHub handle, and organization below:

<!-- prettier-ignore -->
| Name              | GitHub        | Organization     | Location            |
| :---------------- | :------------ | :--------------- | :------------------ |
| Benjie Gillam     | @Benjie       | Graphile         | Chandler's Ford, UK |
| Yaacov Rydzinski  | @yaacovCR     | Individual       | Neve Daniel, IL     |
| Roman Ivantsov    | @rivantsov    | Microsoft        | Redmond, WA, US     |
| Tim Suchanek      | @timsuchanek  | GraphCDN         | Berlin, DE          |
| Saihajpreet Singh | @saihaj       | The Guild        | Ottawa, ON, CA      |
| Matteo Collina    | @mcollina     | NearForm         | Forlì, IT           |
| Uri Goldshtein    | @urigo        | The Guild        | Tel Aviv, IL        |
| Marc-Andre Giroux | @xuorig       | Netflix          | Montreal, Canada    |
| Praveen Durairaju | @praveenweb   | Hasura           | Bangalore, India    |
| Hugh Willson      | @hwillson     | Apollo           | Ottawa, ON, CA      |
| Michael Staib     | @michaelstaib | ChilliCream Inc. | Zurich, CH          |
| Nicholas DeJaco   | @ndejaco2     | Amazon           | Seattle, WA, US     |
| Rafael Abreu      | @grillorafael | Individual       | Dublin, IE.         |
| Julian Mayorga    | @okjulian     | GraphQLApps      | RS, Brazil          |
| Caleb Thomas      | @linux-jedi   | Meta             | Brooklyn, NY, US    |
| Nathan Chapman    | @nathanchapman | Individual      | Austin, TX, US      |
| Bobbie Cochrane   | @bobbiejc     | StepZen          | Oriental, NC        |
| Chi Chan          | @chikit       | Meta             | San Jose, CA, US    |
| Agata Witkowska   | @agata-wit    | Tyk              | Gdansk, PL          |
| Predrag Gruevski  | @obi1kenobi   | Kensho           | Boston, MA, US      |
| Dariusz Kuc       | @dariuszkuc   | Apollo           | Chicago, IL, US     |
| John Starich      | @JohnStarich  | IBM              | Austin, TX, US      |
| Jason Webb        | @jwebb49      | Intuit           | San Diego, CA, US   |
| Laurin Quast      | @n1rual       | The Guild        | Mannheim, DE        |
| Jens Neuse        | @jensneuse    | WunderGraph      | Bretten, DE         |
| Dustin Deus       | @starptech    | WunderGraph      | Wuppertal, DE       |
| Martijn Walraven  | @martijnwalraven | Apollo        | Amsterdam, NL       |
| Jonny Green       | @jonnydgreen  | Unity            | Bath, UK            |
| Daniel Winter     | @d-winter     | Hygraph          | Gießen, DE          |
| Jonas Faber       | @flexzuu      | Hygraph          | Butzbach, DE        |
| Aleksandar Susnjar| @aleksandarsusnjar |             | Markham, ON, Canada |

# GraphQL.NET Connections

## Introduction

GraphQL was created by Facebook and later open sourced. GraphQL represents your data as a queryable graph. The main benefits are:
- GraphQL provides you a single endpoint that allows you to interact with types and fields. Don't focus on how to use multiple endpoints (and multiple requests).
- GraphQL allows you to specify the data you need and nothing more than that. The consumer drives the representation.
Some of the challenges with GraphQL are:
- Caching, and specifically output caching, is more complex.
- The modeling effort needed for using GraphQL might not be worth the investment on simple data models. 
- The build and use of a GraphQL service comes with the complexities of a fairly rich query language.
GraphQL supports queries, mutations and subscriptions. Subscriptions enable a server to send data to clients on a specific event.

We won't go into a comparison with for example REST APIs, but in general GraphQL is worth consideration when:
- A complex data model.
- Consumers that require rich interactions with the data (filtering of data, model-based queries).
- The subscription model provides desired functionality.

<a href="https://graphql.org/learn/">The GraphQL site</a> provides a great learning resource. A GraphQL service is created by defining types and fields on those types.  There are two special types in a GraphQL service: a required Query type and an optional Mutation type. These types are the same as a regular object type, but they are special because they define the entry point of every GraphQL query. In our example below we will introduce a widget type with two fields and define a schema that provides the required query type.

## Collections

<a href="https://graphql-dotnet.github.io">GraphQL DotNet</a> provides documentation and samples that cover a variety of scenarios. One of the more complex scenarios deals with <A href="https://relay.dev/graphql/connections.htm">connections</a>. A major use case for connections is a standardized solution for cursor-based paging through a collection. Paging is commonly provided so the consumer can efficiently interact with larger result sets. GraphQL connections provide the following benefits:
- The ability to paginate through the list.
- The ability to ask for information about the connection itself, like totalCount or pageInfo.
- The ability to ask for information about the edge itself, like cursor or friendshipTime.
- The ability to change how our backend does pagination, since the user just uses opaque cursors.
A connection will return the total count, the page information and a set of edges, each edge existing out of a cursor and an entity.


The <a href="https://github.com/graphql-dotnet/relay">Relay project</a> provides an out-of-the-box solution for collections when the dataset is in memory. In practice however, you will not have the full dataset in memory and want to efficiently query a persistent data store. GraphQL does not provide a standard implementation for interacting with persistent data stores. When implementing the data store queries you will need to ensure that you meet the connections specification.

```csharpTime 
    public class WidgetType : ObjectGraphType<Widget> { 
        public WidgetType()
        {            
            Field<NonNullGraphType<LongGraphType>>("id")
                .Description("The widget identifier.")
                .Resolve(ctx=>
                ctx.Source.Id);
            Field<NonNullGraphType<StringGraphType>>("name")
                .Description("The name of the widget.")
                .Resolve(ctx=>ctx.Source.Name);
        }
    }

     public class WidgetSchema : Schema
    {
        public WidgetSchema(IServiceProvider serviceProvider) : base(serviceProvider)
        {
            Query = new WidgetQuery();
        }
    }


        public class WidgetQuery : ObjectGraphType<Widget>
    {
        public WidgetQuery()
        {
            Name = "Query";
           Connection<WidgetType>()
               .Name("widgets")
               .Returns<Connection<Widget>>()
               .Bidirectional()
               .Argument<DecimalGraphType>("minPrice", "minimum price")
                // .Arguments(
                // new QueryArguments(
                //    new QueryArgument<DecimalGraphType> { Name = "minPrice", Description = "minimum price" }
                //))     
                .PageSize(3)
                .Resolve()
                .WithScope()
                .WithService<IRepository>()                
                .ResolveAsync(ResolveConnectionAsync);
        }

        //see https://github.com/Dotnet-Boxed/Templates/blob/master/Source/GraphQLTemplate/Source/GraphQLTemplate/Schemas/QueryObject.cs for an example.
        private static async Task<Connection<Widget>> ResolveConnectionAsync(
            IResolveConnectionContext<object> context,
   IRepository repository)
        {          
            var first = context.First;
            var afterCursor = Cursor.FromCursor<long?>(context.After);
            var last = context.Last;
            var beforeCursor = Cursor.FromCursor<long?>(context.Before);
            var cancellationToken = context.CancellationToken;
            var pageSize = context.PageSize;          
            var widgets = //collection of widgets to return in ascending order (by cursor)
            var totalCount = //total number of widgets in the data store
            var hasNextPage =  //determine if there are additional widgets after the returned collection of widgets.
            var hasPreviousPage = //determine if there are additional widgets before the returned collection of widgets.
            var (firstCursor, lastCursor) = Cursor.GetFirstAndLastCursor(widgets, x => x.Id);

            return new Connection<Widget>()
            {
                Edges = widgets
                    .Select(widget =>
                        new Edge<Widget>()
                        {
                            Cursor = Cursor.ToCursor(widget.Id),
                            Node = widget,
                        })
                    .ToList(),
                PageInfo = new PageInfo()
                {
                    HasNextPage = hasNextPage,
                    HasPreviousPage = hasPreviousPage,
                    StartCursor = firstCursor,
                    EndCursor = lastCursor,
                },
                TotalCount = totalCount
            };
        }
        
            public static class Cursor
    {
        private const string Prefix = "arrayconnection";

        public static T? FromCursor<T>(string? cursor)
        {
            if (string.IsNullOrEmpty(cursor))
                return default;

            string decodedValue;
            try
            {
                decodedValue = Base64Decode(cursor);
            }
            catch (FormatException)
            {
                return default;
            }

            var prefixIndex = Prefix.Length + 1;
            if (decodedValue.Length <= prefixIndex)
                return default;

            var value = decodedValue.Substring(prefixIndex);
            return (T)Convert.ChangeType(value, typeof(T), CultureInfo.InvariantCulture);
        }

        public static (string? firstCursor, string? lastCursor) GetFirstAndLastCursor<TItem, TCursor>(
            IEnumerable<TItem> enumerable,
            Func<TItem, TCursor> getCursorProperty)
        {
           if (enumerable.Count() == 0)
                return (default,default);

            var firstCursor = ToCursor(getCursorProperty(enumerable.First()));
            var lastCursor = ToCursor(getCursorProperty(enumerable.Last()));
            return (firstCursor, lastCursor);
        }

        public static string ToCursor<T>(T value)
        {
            if (value == null)
                throw new ArgumentNullException(nameof(value));

            return Base64Encode(string.Format(CultureInfo.InvariantCulture, "{0}:{1}", Prefix, value));
        }

        private static string Base64Decode(string value) => Encoding.UTF8.GetString(Convert.FromBase64String(value));

        private static string Base64Encode(string value) => Convert.ToBase64String(Encoding.UTF8.GetBytes(value));
    }
```

Note that GraphQL implements cursor-based pagination. The cursors are opaque, either offset or ID-based pagination can be implemented.
The <a href="https://relay.dev/graphql/connections.htm">specification</a> states in particular that:
- The ordering of edges should be the same when using first/after as when using last/before, all other arguments being equal.
- hasPreviousPage is used to indicate whether more edges exist prior to the set defined by the clients arguments. If the client is paginating with last/before, then the server must return true if prior edges exist, otherwise false. If the client is paginating with first/after, then the client may return true if edges prior to after exist, if it can do so efficiently, otherwise may return false. 
- hasNextPage is used to indicate whether more edges exist following the set defined by the clients arguments. If the client is paginating with first/after, then the server must return true if further edges exist, otherwise false. If the client is paginating with last/before, then the client may return true if edges further from before exist, if it can do so efficiently, otherwise may return false.
To shorten execution time, you can utilize multi-threading or multiple queries in a single round-trip as your repository allows.

# How the new Community site was built

Talk by [Javier Gamarra](https://twitter.com/nhpatt) for [/dev/24](https://liferay.dev/24) about the new Liferay Community Site (also called Questions), built using React + GraphQL.

## So... where is it?

Ready to be launched [soon™]

## How does it look?

[Let's see...](https://dev.liferay.dev/ask#/)

## Why a new site?

Personal opinions follow:

* We want **more participation of Liferay product teams** in the forums
* We need to improve feature request flow, listen **more** to the community
* We want to ease interaction and **participation**
* We need to do **dog-fooding** of our APIs

## So how to we build it?

* React... _Why?_
    * It started as a create-react app
* GraphQL... _Why?_
* Using Liferay MB APIs (MBMessage, MBThread...)

## Talk about

1. New view for MBoards
	1.  Will be launched as such in the future (but hey, Open Source, it's there in the [repo](https://github.com/liferay/liferay-portal/tree/master/modules/apps/questions/questions-web)).
	
1. Questions is a portlet*
	1. Just configuration, deal with ranks and create the URL for the file ItemSelector.
	1. Served inside Liferay with existing fetch -> no auth/CSRF management

1. Headless use
    1. Uses permissions to avoid client permissions   
    
    ```jsx
    {answer.actions['reply-to-message'] && (
        <ClayButton
            className="text-reset"
            displayType="unstyled"
            onClick={() => setShowNewComment(true)}
        >
            {Liferay.Language.get('reply')}
        </ClayButton>
    )}
    ``` 
   
   1. Heavy use of GraphQL (around 40 queries, with fragments)*
   1. Heavy use of GraphQL relationships
   
   ```
    export const getMessagesQuery = gql`
    	query messageBoardThreadMessageBoardMessages(
    		$messageBoardThreadId: Long!
    		$page: Int!
    		$pageSize: Int!
    		$sort: String!
    	) {
    		messageBoardThreadMessageBoardMessages(
    			messageBoardThreadId: $messageBoardThreadId
    			page: $page
    			pageSize: $pageSize
    			sort: $sort
    		) {
    			items {
    				actions
    				aggregateRating {
    					ratingAverage
    					ratingCount
    					ratingValue
    				}
    				articleBody
    				creator {
    					id
    					image
    					name
    				}
    				creatorStatistics {
    					joinDate
    					lastPostDate
    					postsNumber
    					rank
    				}
    				encodingFormat
    				friendlyUrlPath
    				id
    				messageBoardMessages {
    					items {
    						actions
    						articleBody
    						creator {
    							id
    							image
    							name
    						}
    						encodingFormat
    						id
    						showAsAnswer
    					}
    				}
    				myRating {
    					ratingValue
    				}
    				showAsAnswer
    			}
    			pageSize
    			totalCount
    		}
    	}
    `;
    ```
   
   1. Including _parent_ relationships
   
   ```
   export const getSectionBySectionTitleQuery = gql`
   	query messageBoardSections($filter: String!, $siteKey: String!) {
   		messageBoardSections(
   			filter: $filter
   			flatten: true
   			pageSize: 1
   			siteKey: $siteKey
   			sort: "title:asc"
   		) {
   			actions
   			items {
   				actions
   				id
   				messageBoardSections(sort: "title:asc") {
   					actions
   					items {
   						id
   						description
   						numberOfMessageBoardSections
   						numberOfMessageBoardThreads
   						parentMessageBoardSectionId
   						subscribed
   						title
   					}
   				}
   				numberOfMessageBoardSections
   				parentMessageBoardSection {
   					id
   					messageBoardSections {
   						items {
   							id
   							numberOfMessageBoardSections
   							parentMessageBoardSectionId
   							subscribed
   							title
   						}
   					}
   					numberOfMessageBoardSections
   					parentMessageBoardSectionId
   					subscribed
   					title
   				}
   				parentMessageBoardSectionId
   				subscribed
   				title
   			}
   		}
   	}
   `;
   ```
   
    1. We use MB* APIs + Ratings + Subscriptions
    
1. API Changes
    1. No extension of APIs (remember that GraphQL APIs are inferred from REST + extended)
        1. We can add our API, and endpoint inside an existing API, change an existing endpoint, change a entity to add or remove fields, remove an endpoint...
        1. +do it only for GraphQL
        
        ```java
        @GraphQLTypeExtension(Document.class)
        public class GetDocumentFolderTypeExtension {
        
            public GetDocumentFolderTypeExtension(Document document) {
                _document = document;
            }
        
            @GraphQLField(description = "Retrieves the document folder.")
            public DocumentFolder folder() throws Exception {
                return _applyComponentServiceObjects(
                    _documentFolderResourceComponentServiceObjects,
                    Query.this::_populateResourceContext,
                    documentFolderResource ->
                        documentFolderResource.getDocumentFolder(
                            _document.getDocumentFolderId()));
            }
        
            private Document _document;       
        }  
        ```
       
        ```java
        public interface GraphQLContributor {
        
        	public default Object getMutation() {
        		return null;
        	}
        
        	public String getPath();
        
        	public default Object getQuery() {
        		return null;
        	}
        
        }
        ```
    1. New APIs: ranked information
   
1. Uses Apollo JS Client with hooks (we were using a custom client before)

    ```javascript
    const {data, loading} = useQuery(getUserActivityQuery, {
        variables: {filter: `creatorId eq ${creatorId}`, page, pageSize, siteKey,},
    });
    ```

    1. Not ideal
    1. Hooks move data transformation & variables to component
    1. A bit verbose
    1. Refetch + onComplete is wonky
    1. Hard to deal with conditional logic
    1. Cache is hit&miss, too many manual invalidation
    1. But I would say that is worth it, just for the times the syntax is brief and local cache alone
    1. Mixed usage with Apollo client directly
    1. Mixed/continuation calls are ugly so we avoid them:
    ```javascript
    const [deleteMessage] = useMutation(deleteMessageQuery, {
        onCompleted() {
            if (comments && comments.length) {
                Promise.all(
                    comments.map(({id}) =>
                        client.mutate({
                            mutation: deleteMessageQuery,
                            variables: {messageBoardMessageId: id},
                        })
                    )
                ).then(() => {
                    deleteAnswer(answer);
                });
            }
            else {
                deleteAnswer(answer);
            }
        },
    });
    ```
   
1. Works with old forum	
	1.  Compatible with old data and has to render a large forum (500.000? threads)
	1.  Should-be compatible with BBCode and HTML but we are going the HTML way (conversion!)
	1.  Support for old URLs with urlrewrite.xml + redirect components
	```xml
    <rule>
        <from>^/web/guest/community/forums/message_boards(.{0,2000})$</from>
        <to type="permanent-redirect">%{context-path}/web/guest/community/forums/-/message_boards$1</to>
    </rule>
	```
	1.  +slugs for everything
	1.  Uses CKEditor (conversion!). ~~Very~~ thin layer over _Editor_ in _frontend-editor-ckeditor-web_ (that is a very thin layer over CKEditor 4) + itemselector plugin.
	
    ```javascript
    import CKEditor from 'ckeditor4-react';
    import React, {useEffect} from 'react';
    
    const BASEPATH = '/o/frontend-editor-ckeditor-web/ckeditor/';
    
    const Editor = React.forwardRef((props, ref) => {
        useEffect(() => {
            Liferay.once('beforeScreenFlip', () => {
                if (
                    window.CKEDITOR &&
                    Object.keys(window.CKEDITOR.instances).length === 0
                ) {
                    delete window.CKEDITOR;
                }
            });
        }, []);
    
        return <CKEditor ref={ref} {...props} />;
    });
    
    CKEditor.editorUrl = `${BASEPATH}ckeditor.js`;
    window.CKEDITOR_BASEPATH = BASEPATH;
    
    export {Editor};
    ```
	
	1.  Uses Highlight JS
	
1. Performance
    1. We are reviewing all the endpoints
    1. Elastic issues with big size windows... UX makes sense? || scrolling API || DB 
    1. After small tuning (1 day) everything looks good (<1 sec load)
    1. NestedFields... future lazy implementation?
    
1. Styles
    1. Some new UI patterns like breadcrumbs or lists
    1. But Clay everywhere (FTW!)

## Tech debt

* Very hacky implementation for subscriptions URLs
* Some limitations regarding rankings APIs because DB structure
* Use fragments in GraphQL calls !
* ~~MBCategory~~MessageBoardSection management is messy
* Add more friendlyURLs
* Better use of related queries
* Review accesibility :(

## Questions?

* ...
* ...
* ...

**Thanks!**
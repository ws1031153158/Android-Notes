XML                          Compose  
─────────────────────────────────────────────────  
LinearLayout(vertical)   →   Column  
LinearLayout(horizontal) →   Row  
FrameLayout              →   Box  
RecyclerView             →   LazyColumn / LazyRow  
TextView                 →   Text  
Button                   →   Button  
ImageView                →   Image / AsyncImage  
EditText                 →   TextField  
CardView                 →   Card  
View.GONE/VISIBLE        →   if(visible) { ... }  
margin/padding           →   Modifier.padding()  
background               →   Modifier.background()  
onClick                  →   Modifier.clickable {}  
match_parent             →   Modifier.fillMaxWidth()  
wrap_content             →   默认行为  

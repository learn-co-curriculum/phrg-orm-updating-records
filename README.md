# orm-updating-records

SONG. Bring in the UPDATE method now. So you can update an existing Artist. problem is save still creates new ones. so refactor that to an insert method and how do you know UPDATE vs. INSERT? gotta see if it has already persisted. How do you know that? check if id is filled out. Aslo refactor #save into #find_or_create_by
